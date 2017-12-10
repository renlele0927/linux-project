# keepalived+lvs(DR)

### 系统环境:**rhel6.5 samll install ,iptables and selinux disabled**
### 主机规划:

Name | IP | ROLE |software
-| :-: |:-:|-
server1 | 172.25.4.1 | VS(Virtual Server) | ipvsadm,keepalived,httpd  
server2 | 172.25.4.2 | VS(Virtual Server) | ipvsadm,keepalived,httpd
server3 | 172.25.4.3 | RS(Real Server) | arptables,httpd
server4 | 172.25.4.4 | RS(Real Server) | arptables,httpd

VIP:172.25.4.100

### keepalived功能:
    keepalived的作用是检测服务器的状态，如果有一台web服务器宕机，或工作出现故障，
    Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该
    服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中，这些
    工作全部自动完成。

keepalived软件包下载: 

[下载地址](http://www.keepalived.org/download.html)

>**本次实验使用的是keepalived-1.2.20**

[root@server1 ~]# tar zxf keepalived-1.2.20.tar.gz

## VS配置:
(server1,server2)

server1:

### 1.软件安装：

    [root@server1 ~]# tar zxf keepalived-1.2.20.tar.gz
    [root@server1 ~]# yum install gcc openssl-devel -y
    [root@server1 ~]# cd keepalived-1.2.20
    [root@server1 keepalived-1.2.20]# ./configure --prefix=/usr/local/keepalived
    [root@server1 keepalived-1.2.20]# make
    [root@server1 keepalived-1.2.20]# make install

### 2.keepalived文件配置:

    # ln -s /usr/local/keepalived/etc/sysconfig/keepalived  /etc/sysconfig/
    # ln -s /usr/local/keepalived/etc/keepalived  /etc
    # ln -s /usr/local/keepalived/etc/rc.d/init.d/keepalived  /etc/init.d/
    # chmod +x /etc/init.d/keepalived
    # ln -s /usr/local/keepalives/sbin/keepalived /sbin/

**vim /etc/keepalived/keepalived.conf**

    ! Configuration File for keepalived

    global_defs {
    notification_email {
     root@localhost     ##接受警报的email地址,可以添加多个
    }
    notification_email_from root@localhost      ##设置邮箱的发送地址
    smtp_server 127.0.0.1       ##设置 smtp server 地址
    smtp_connect_timeout 30     ##设置连接smtp服务器超时时间
    router_id LVS_DEVEL         ##load balancer 的标识 ID,用于 email 警报
    }

    vrrp_instance VI_1 {
    state MASTER    ##备机改为 BACKUP,此状态是由 priority 的值来决定的,当前priority 的值小于备机的值,那么将会失去 MASTER 状态
    interface eth0  ##HA 监测网络接口
    virtual_router_id 51    ##主、备机的 virtual_router_id 必须相同,取值 0-255
    priority 150    ##主机的优先级,备份机改为 50,主机优先级一定要大于备机
    advert_int 1    ##主备之间的通告间隔秒数
    authentication {    ##主备切换时的验证
        auth_type PASS  ##设置验证类型,主要有 PASS 和 AH 两种
        auth_pass 1111  ##设置验证密码,在一个 vrrp_instance 下,MASTER 与 BACKUP 必须使用相同的密码才能正常通信
        }
    virtual_ipaddress {
        172.25.4.100    ##设置虚拟 IP 地址,可以设置多个虚拟 IP 地址,每行一个
        }
    }

    virtual_server 172.25.4.100 80 {    ##定义虚拟服务器
    delay_loop 6        ##每隔 6 秒查询 realserver 状态
    lb_algo rr          ##lvs 调度算法,这里使用轮叫,一共有10种
    lb_kind DR          ##LVS 是用 DR 模式,还有，NAT，TUN，FullNAT模式
    protocol TCP        ##指定转发协议类型,有 tcp 和 udp 两种
    real_server 172.25.4.3 80 {     ##定义真实主机
        weight 1        ##配置服务节点的权值,权值大小用数字表示,数字越大,权值越高,设置权值的大小可以为不同性能的服务器分配不同的负载,可以对性能高的服务器设置较高的权值,而对性能较低的服务器设置相对较低的权值,这样就合理的利用和分配了系统资源
        TCP_CHECK {     ##realserve 的状态检测设置部分,单位是秒
            connect_timeout 1       ##3 秒无响应超时
            nbI_get_retry 1         ##重试次数
            delay_before_retry 1    ##重试间隔
            }
      }
     real_server 172.25.4.4 80 {
        weight 1
        TCP_CHECK {
        connect_timeout 1
        nbI_get_retry 1
        delay_before_retry 1
        }
      }
    }

**注意:**

    VS只之间是有主备区分的 ,区分之处在于下面这两处 
    主机 state 是MASTER 
    备机 state 是BACKUP 
    主机 priority 值一定要比备机高

    # /etc/init.d/keepalived start      ##启动keepalive

*有邮件提醒:*因为RS还没有进行配置,所以我们会收到来自keepalived的邮件,提示我们检测不到真正主机,不过不影响,只要keepalived能正常启动,就说明我们配置已经成功了

### 3.安装ipvsadm:

1.首先需要配置yum源

    [root@server1 ~]# cat /etc/yum.repos.d/rhel6.5.repo  
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
2.直接用yum安装：

    # yum install -y ipvsadm

3.查看keepalive状态：

    [root@server1 keepalived-1.2.20]# ipvsadm -ln
    IP Virtual Server version 1.2.1 (size=4194304)
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  172.25.4.100:80 rr
      -> 172.25.4.3:80                Route   1      0          0         
      -> 172.25.4.4:80                Route   1      0          0 

server2:

    这台主机作为vs备机,将安装包拷贝过来,加上软链接,修改keepalive.conf配置文件中,state BACKUP,priority 50,最后安装ipvsadm即可。

### RS配置
(server3,server4)

server3:

    # ip addr add 172.25.4.100/24 dev eth0    ##临时添加VIP，需要开始手动开启，若需要开机自动添加，则修改/etc/rc.d/rc.local
    # yum install arptables_jf -yum     ##安装arptables
    # yum install -y httpd              ##安装apache
    # echo server3 > /var/www/html/index.html   ##创建测试页面
    # /etc/init.d/httpd start           ##打开apache服务
    # 配置arp策略
    # arptables -A IN -d 172.25.4.100 -j DROP   ##拒绝网络里对于vip的arp请求
    # arptables -A OUT -s 172.25.4.100 -j mangle --mangle-ip-s 172.25.4.3

server4:

    # ip addr add 172.25.4.100/24 dev eth0    ##临时添加VIP，需要开始手动开启，若需要开机自动添加，则修改/etc/rc.d/rc.local
    # yum install arptables_jf -yum     ##安装arptables
    # yum install -y httpd              ##安装apache
    # echo server4 > /var/www/html/index.html   ##创建测试页面
    # /etc/init.d/httpd start           ##打开apache服务
    # 配置arp策略
    # arptables -A IN -d 172.25.4.100 -j DROP   ##拒绝网络里对于vip的arp请求
    # arptables -A OUT -s 172.25.4.100 -j mangle --mangle-ip-s 172.25.4.4

## 测试:

1.高可用测试:停止 master 上的 keepalived 服务,看 backup 是否接管

2.负载均衡测试:

    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server4
    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server3
    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server4
    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server3

3.故障切换测试:任意关闭 realserver 上的 httpd 服务,Keepalived 监控模块是否能及时发现,然后**屏蔽故障节点**,同时将服务转移到正常节点来执行。

    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server4
    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server4
    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server4
    [kiosk@foundation4 Desktop]$ curl  172.25.4.100
    server4

4.打开server3上的httpd，**自动添加**，并继续负载均衡。
