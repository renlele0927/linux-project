# Keepalived

## 简介：

    keepalived是一个免费开源的，用C语言编写的，类似于layer3, 4 & 7交换机制的软件，也就是我们平时说的第3层、第4层和第7层交换。Keepalived是自动完成，不需人工干涉。
    Keepalived的作用是检测服务器的状态，如果有一台web服务器宕机，或工作出现故障，Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的服务器。
    主要提供loadbalancing（负载均衡）和 high-availability（高可用）功能，负载均衡实现需要依赖Linux的虚拟服务内核模块（ipvs），而高可用是通过VRRP协议实现多台机器之间的故障转移服务。

## 工作原理：

    Layer3,4&7工作在IP/TCP协议栈的IP层，TCP层，及应用层,原理分别如下：
    Layer3：Keepalived使用Layer3的方式工作式时，Keepalived会定期向服务器群中的服务器发送一个ICMP的数据包（既我们平时用的Ping程序）,如果发现某台服务的IP地址没有激活，Keepalived便报告这台服务器失效，并将它从服务器群中剔除，这种情况的典型例子是某台服务器被非法关机。Layer3的方式是以服务器的IP地址是否有效作为服务器工作正常与否的标准。
    Layer4:如果您理解了Layer3的方式，Layer4就容易了。Layer4主要以TCP端口的状态来决定服务器工作正常与否。如web server的服务端口一般是80，如果Keepalived检测到80端口没有启动，则Keepalived将把这台服务器从服务器群中剔除。
    Layer7：Layer7就是工作在具体的应用层了，比Layer3,Layer4要复杂一点，在网络上占用的带宽也要大一些。Keepalived将根据用户的设定检查服务器程序的运行是否正常，如果与用户的设定不相符，则Keepalived将把服务器从服务器群中剔除。

![1](http://mdpicture-1253499256.file.myqcloud.com/Keepalived/keepalived1.png)

**上图是Keepalived的功能体系结构，大致分两层：用户空间（user space）和内核空间（kernel space）。**

内核空间：主要包括IPVS（IP虚拟服务器，用于实现网络服务的负载均衡）和NETLINK（提供高级路由及其他相关的网络功能）两个部分。

用户空间：

    WatchDog：负载监控checkers和VRRP进程的状况。
    VRRP Stack：负载负载均衡器之间的失败切换FailOver，如果只用一个负载均衡器，则VRRP不是必须的。
    Checkers：负责真实服务器的健康检查healthchecking，是keepalived最主要的功能。换言之，可以没有VRRP Stack，但健康检查healthchecking是一定要有的。
    IPVS wrapper：用户发送设定的规则到内核ipvs代码。
    Netlink Reflector：用来设定vrrp的vip地址等。

**Keepalived的所有功能是配置keepalived.conf文件来实现的。**
