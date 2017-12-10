# RHCS配置


## 实验环境：iptables and selinux disabled**(必须保证)**

Name | IP |software
-| :-: | -
server1 | 172.25.4.1 | luci,ricci  
server2 | 172.25.4.2 | ricci
foundation4 | 172.25.4.250 | fence_virtd

VIP:172.25.4.100


## 各部分功能介绍：

1.集群中luci的作用：luci是用来配置和管理集群的，监听在8084端口上。

2.集群中ricci的作用：ricci是安装在后端的每个节点上的，luci管理集群上各节点，就是通过和节点上的ricci进行通信的，ricci监听在1111端口上。

3.集群中fence的作用：在HA集群中，备份服务器B通过心跳线来发送数据包来看服务器A是否活着，假设主服务器A接受了大量的客户端访问请求，服务器A的CPU负载达到了100%还响应不过来，资源已经耗尽，没有办法回复服务器B的数据包响应（或者数据包延迟），这时服务器B认为服务器A已经挂了，于是备份服务器B把资源争夺过来，自己做主服务器，过了一段时间服务器A响应过来了，服务器A觉得自己是主服务器，服务器B觉得自己是主服务器，它们之间开始抢夺资源，集群资源被多个节点占有，两个服务器同时向资源写数据，破化了安全性和数据一致性，这种情况的发生就叫做“脑裂”。服务器A负载过重，响应不过来了，有了fence机制，fence会自动的把服务器A给fence掉，组织了“脑裂”的发生。

### fence的工作原理：
    当意外原因导致主机异常，或者宕机时，备机首先会调用fence设备，然后通过fence设备将异常主机重启
    或者从网络隔离，当fence操作成功执行后，返回信息给备机，备机在接到fence成功的信息后，开始接管
    主机的服务和资源。这样通过fence设备，将异常节点占据的资源进行释放，保证资源和服务始终运行在一
    个节点上。

### fence的分类：
>硬件fence：电源fence，通过关掉电源来踢掉故障服务器。

>软件fence：fence卡（智能卡），通过线缆，软件来踢掉故障服务器。

>内部fence：IBM RSAII卡，HP的iLO，IPMI的设备等。

>外部fence：UPS,SAN SWITCH,NETWORK SWITCH等。

**实际环境中，fence卡连接的都是专线，使用专用的fence网卡，不会占用数据传输线路，这样更能保证稳定性及可靠性。fence卡的IP网络和集群网络相互依存，即可以互通。**


## 开始配置：
(server1,server2)
### 1.yum源配置：
    # vim /etc/yum.repos.d/rhel6.5.repo 
    [rhel6.5-source]
    name=rhel6.5 $releasever - $basearch - Source
    baseurl=http://172.25.254.4/rhel6.5/Server
    enabled=1
    gpgcheck=1
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

    [HighAvailability]
    name=HighAvailability
    baseurl=http://172.25.4.250/rhel6.5/HighAvailability
    gpgcheck=0

    [LoadBalancer]
    name=LoadBalancer
    baseurl=http://172.25.4.250/rhel6.5/LoadBalancer
    gpgcheck=0

    [ResilientStorage]
    name=ResilientStorage
    baseurl=http://172.25.4.250/rhel6.5/ResilientStorage
    gpgcheck=0

    [ScalableFileSystem]
    name=ScalableFileSystem
    baseurl=http://172.25.4.250/rhel6.5/ScalableFileSystem
    gpgcheck=0


#### 将此文件拷贝到另外一台主机上:
    # scp /etc/yum.repos.d/rhel6.5.repo root@172.25.4.2:/etc/yum.repos.d/ 

### 安装软件：
1.ricci（每个节点都需要）

    # yum install -y ricci

    # passwd ricci  ##设置ricci用户密码

    # /etc/init.d/ricci start   ##启动
    # chkconfig ricci on        ##设置开机自启动

2.luci：
只在一台主机（172.25.4.1）

    # yum install -y luci

    # /etc/init.d/luci start     ##启动
    # chkconfig luci on      ##设置开机自启动

#### 通过浏览器登陆到管理界面进行配置

>https://172.25.4.1:8084

    帐号：root
    密码：******   （luci所在服务器超级用户密码）
    *警告直接ok

    1.创建cluster，NodeName 写节点主机名，密码就是安装后设置的ricci用户密码。
    如图：

![1](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS1.png)

    2.创建完成后，用clustat命令可以查看集群中每个节点以及服务的运行状态。
    -i 3 :表示每三秒查看一次。
    online:表示在线；
    offline：表示不在线；
    local：指的是在本地；
    started：表示已经运行；
    stopping：停止运行；
    failed：运行失败。

3.安装fence_virt服务（不可缺少）：
只在物理机（172.25.254.4）

    # yum install -y fence-virtd-multicast fence-virtd-serial fence-virtd
    （补充使用：如果不能正常工作，继续安装：libxshmfence fence-virtd-libvirt fence-virt libxshmfence）

    # fence_virtd -c        ##交互式配置fence文件

    Module search path [/usr/lib64/fence-virt]: 

    Available backends:
    libvirt 0.1
    Available listeners:
    multicast 1.2
    serial 0.4

    Listener modules are responsible for accepting requests
    from fencing clients.

    Listener module [multicast]:        ##组播模式

    The multicast listener module is designed for use environments
    where the guests and hosts may communicate over a network using
    multicast.

    The multicast address is the address that a client will use to
    send fencing requests to fence_virtd.

    Multicast IP Address [225.0.0.12]:  ##组播地址

    Using ipv4 as family.

    Multicast IP Port [1229]:       ##组播默认端口

    Setting a preferred interface causes fence_virtd to listen only
    on that interface.  Normally, it listens on all interfaces.
    In environments where the virtual machines are using the host
    machine as a gateway, this *must* be set (typically to virbr0).
    Set to 'none' for no interface.

    Interface [br0]:    ##网卡设备，我这里时网桥名称，注意根据自己情况修改

    The key file is the shared key information which is used to
    authenticate fencing requests.  The contents of this file must
    be distributed to each physical host and virtual machine within
    a cluster.

    Key File [/etc/cluster/fence_xvm.key]:      ##key文件默认路径

    Backend modules are responsible for routing requests to
    the appropriate hypervisor or management layer.

    Backend module [libvirt]:   ##模块基于那个协议

    Configuration complete.

    === Begin Configuration ===
    backends {
    libvirt {
        uri = "qemu:///system";
      }
    }

    listeners {
    multicast {
        port = "1229";
        family = "ipv4";
        interface = "br0";
        address = "225.0.0.12";
        key_file = "/etc/cluster/fence_xvm.key";
      }
    }

    fence_virtd {
        module_path = "/usr/lib64/fence-virt";
        backend = "libvirt";
        listener = "multicast";
    }

    === End Configuration ===
    Replace /etc/fence_virt.conf with the above [y/N]? y


>fence_xvm.key:fence卡的key文件，名字不要修改，fence_xvm是fence卡的一种类型。

>/etc/cluster/fence_xvm.key:这个目录需要手动创建。

    # mkdir /etc/cluster
    然后生成key：
    # dd if=/dev/urandom of=/etc/cluster/fence_xvm.key bs=128 count=1
    最后将key文件发送到各节点中。
    # scp /etc/cluster/fence_xvm.key root@ip:/etc/cluster/fence_xvm.key

    # systemctl start fence_virtd.service   ##启动fence

    登录到集群管理界面，并添加fence设备（server1和server2都需要添加）：

    每个节点的/etc/cluster/cluster.conf需要同步，所以每次点击submit都会缓冲一段时间。
    给每个节点添加fence,如图：

![2](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS2.png)

![3](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS3.png)

    添加fence设备时，Domain那一栏写可以区别各个节点的内容，可以是主机名，也可以是唯一标示符，为了
    精准建议选择UUID，我用的是虚拟机UUID。

## 测试：

在server1上运行：

    # fence_node server2    ##若显示fence success则成功。如果失败，注意iptable,
                            selinux是否关闭，各点key是否一致。否则可能因为无法通信而失败。

**至此，RHCS集群基本要求全部满足了。**

Failover Domains:

    顾名思义，Failover Domaims就是失败率，优先级的意思。在集群中，有个问题就是1机器先启动服务还
    是2机器先启动服务，通过设置FailoverDomains来解决，数字越低，优先级越高，这个机器就先启动服。
如图：

![4](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS4.png)

### 创建资源：
1.先创建一个VIP资源，如图：

![5](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS5.png)

2.创建apache资源

![6](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS6.png)

3.添加Service Group：

![7](http://mdpicture-1253499256.file.myqcloud.com/RHCS/RHCS7.png)

#### 在每个节点上面配置httpd服务

    # yum install -y httpd
    # echo server1 >/var/www/html/index.html

*注意不要手动打开httpd*

## 测试：
1.httpd服务本来在server1上运行，我们关掉server1上的httpd,看httpd是否迁移到server2，而且VIP也迁移了。

2.如果server2挂了，看是否会被fence掉。

    # echo c> /proc/sysrq-trigger   ##手动server2的内核发生故障，看服务是否迁移成功。
