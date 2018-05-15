#<center>使用IFB进行Ingress流量控制
----
##流量控制
Linux的Traffic Control（以下简称TC）是linux内核中内置的流量控制模块，通过该模块可以进行网络端口的收发流量控制，包括流量限速、流量整形、模拟丢包等功能。TC主要通过三种元素——qdisc、class、filter组成的树状结构来管理多种流量控制规则。
####QDISC
QDISC即队列规则，TC通过各种不同的队列规则来实现数据包的收发控制。队列规则主要分为两大类——可分类队列和不可分类队列。可分类的队列通过filter（或ToS、priority）的过滤后可以分为不同的class，从而为每个class单独配置队列规则。与之相反，不可分类队列只能应用在叶子结点，无法继续分类。
######可分类队列
+ cbq：       class based queueing，基于类别的队列
+ htb：       hierarchy token bucket，分层令牌桶
+ prio：      优先队列
######不可分类队列
+ [p|b]fifo： 以包或字节为单位的先入先出队列
+ pfifo_fast：默认的先入先出队列
+ tbf：       token bucket filter，令牌桶过滤器
+ red：       random early detection，随机早期检测
+ sfq：       stochastic fairness queuing，随机公平队列
####CLASS
类(Class)组成一个树，每个类都只有一个父类，而一个类可以有多个子类。某些qdisc(例如：cbq和htb)允许在运行时动态添加类，而其它的qdisc(例如：prio)不允许动态建立类。
允许动态添加类的qdisc可以有零个或者多个子类，由它们为数据包排队。
此外，每个类都有且仅有一个叶子qdisc。默认情况下，这个叶子qdisc使用pfifo_fast的方式排队。我们也可以使用其它类型的qdisc代替这个默认的qdisc，而且这个qdisc可以再次分类。
####FILTER
过滤器（filter）通过匹配数据包头部的域或iptable的标记对数据包进行分类，将数据包分发给匹配的子类。如果所有子类都不匹配，过滤器会将数据包发送给当前类的叶子qdisc。
除了过滤器以外，有些qdisc还支持ToS（服务类型）分类方式。
####ingress & egress
TC默认的egress根结点是1:0，而ingress结点需要通过如下命令进行添加才可以进行操作。

    //添加ingress结点
    tc qdisc add dev eth0 ingress

ingress结点的标号为ffff:0，这个结点无法添加子类，只能通过过滤器进行限速。因此我们需要ifb来实现对ingress的流量控制。

    //为ingress结点添加过滤器
    tc filter add dev eth0 parent ffff: protocol ip prio 10 \
      u32 match ip src 0.0.0.0/0 \
      police rate 2048kbps burst 1m drop flowid :1

####TC命令实例
    //在根结点添加htb队列，未分类的包默认发送给1:1子类
    tc qdisc add dev eth0 root handle 1:htb default 1
    //以1：为父结点，创建标号为1:1的子类，子类使用htb队列规则，且速率上下限均为40mbps
    tc class add dev eth0 parent 1: classid 1:1 htb rate 40mbit ceil 40mbit
    /*
      protocol ip表示该过滤器应该检查报文分组的协议字段。
      prio 1表示优先级，对于不同优先级的过滤器，系统将按照从小到大的优先级顺序来执行过滤器。
      对于相同的优先级，系统将按照命令的先后顺序执行。
      这个过滤器还用到了u32选择器，判断的是dport字段。
      如果该字段与Oxffff进行与操作的结果是8O，则flowid 1:1表示将把该数据流分配给类别1:1
    */
    tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip dport 80 0xffff flowid 1:1
##IFB
IFB的全称是Intermediate Functional Block，即中间功能块。由于TC只能对ingress流量进行限流，当我们需要对ingress流量进行整形时，就需要借助ifb进行转发。通过将ingress重定向至ifb，再于ifb的egress处添加队列规则，就能够实现对ingress的整形。
此外，在egress处也可以加入一个ifb进行映射，达到同样的配置应用于多块网卡的效果。
######Redirect & Mirror
ifb的两种主要用法分别通过redirect和mirror实现。当我们需要对ingress整形时，采用redirect方式对数据包进行重定向发送，这样避免了原有的入口流量和经过整形后的入口流量重复接收。redirect参数下，该命令会将原有的ingress全部转发至ifb device，而不会发送给最初的接受网卡。
当我们需要监听流量时，就可以采用mirror参数。这种情况下，ingress会被复制一份发送到ifb device以供后续的监测处理，同时不会影响到正常的ingress接收。

    //这条命令可以将所有ingress的ip包重定向或镜像发送到ifb device0
    TC filter add dev eth0 parent ffff: protocol ip prio 1 u32 \
      match u32 0 0 flowid 1:1 \
      action mirred egress [redirect|mirror] dev ifb0
####ifb配置实例
在远程配置ifb时，千万注意在配置完ifb的队列后再重定向ingress，否则可能会因为入口流量被重定向而无法连接远程服务器。

    //加载ifb模块
    modprobe ifb
    //启用虚拟设备ifb
    ip link set dev ifb0 up
    //配置ifb设备的队列，和上文配置网卡方式相同
    TC qdisc add dev ifb0 root handle 1: prio 
    TC qdisc add dev ifb0 parent 1:1 handle 10: sfq
    TC qdisc add dev ifb0 parent 1:2 handle 20: tbf rate 20kbit buffer 1600 limit 3000
    TC qdisc add dev ifb0 parent 1:3 handle 30:sfq
    TC filter add dev ifb0 protocol ip pref 1 parent 1: handle 1 fw classid 1:1
    TC filter add dev ifb0 protocol ip pref 2 parent 1: handle 2 fw classid 1:2
    //将ingress重定向至ifb
    tc filter add dev eth0 parent fff: protocol ip u32 match u32 0 0 flowid 1:1 action mirred egress redirect dev ifb0
######参考资料：
+ [Linux TC(Traffic Control) 简介](https://blog.csdn.net/zhaobryant/article/details/38797655)
+ [tc(8) - Linux manual page](http://www.man7.org/linux/man-pages/man8/tc.8.html)
+ [networking:ifb [Linux Foundation Wiki]](https://wiki.linuxfoundation.org/networking/ifb)
+ [输入方向的流量控制](https://blog.csdn.net/zhangskd/article/details/8240290)
+ [Linux TC的ifb原理以及ingress流控](https://blog.csdn.net/dog250/article/details/40680765)