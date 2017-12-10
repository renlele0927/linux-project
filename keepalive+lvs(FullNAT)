# keepalive+lvs(FullNAT)

### 系统环境:**rhel6.5 samll install ,iptables and selinux disabled**
### 主机规划:

Name | IP | ROLE |software
-| :-: |:-:|-
server1 | 172.25.4.1 | VS(Virtual Server) | ipvsadm,keepalived  
server2 | 172.25.4.2 | VS(Virtual Server) | ipvsadm,keepalived
server3 | 172.25.4.3 | RS(Real Server) | arptables,httpd
server4 | 172.25.4.4 | RS(Real Server) | arptables,httpd

VIP:172.25.4.100


### LVS-FullNat原理：

    LVS当前应用主要采用DR和NAT模式，但这2种模式要求RS和DR在同一个vlan中，导致部署成本过高；
    TUN模式虽然可以跨vlan，但RS上需要部署ipip模块等，网络拓扑上需要连通外网，较复杂，不易运维。
    为了解决上述问题，我们在DR上添加了一种新的转发模式：FullNat，该模式和NAT模式的区别是：
    Packet IN时，除了做DNAT，还做SNAT（用户ip-内网ip），从而实现DR-RS间可以跨vlan通讯，
    RS只需要连接到内网。
![](http://mdpicture-1253499256.file.myqcloud.com/Screenshot%20from%202017-12-10%2020-07-52.png)


## VS配置:
(server1,server2)

**安装带有FullNAT的keepalive时候，首先需要重新编译内核。**
#### 1.编译内核：

    # rpm -ivh kernel-2.6.32-220.23.1.el6.src.rpm   ##warning提示mockbuild用户以及组不存在，不用管。
    # cd rpmbuild/
    # cd SPECS
    # yum install -y rpm-build.x86_64   ##需要安装rpm-build
    # rpmbuild -bp kernel.spec          ##缺少依赖包
    error: Failed build dependencies:
        gcc >= 3.4.2 is needed by kernel-2.6.32-220.23.1.el6.x86_64
        redhat-rpm-config is needed by kernel-2.6.32-220.23.1.el6.x86_64
        patchutils is needed by kernel-2.6.32-220.23.1.el6.x86_64
        xmlto is needed by kernel-2.6.32-220.23.1.el6.x86_64
        asciidoc is needed by kernel-2.6.32-220.23.1.el6.x86_64
        elfutils-libelf-devel is needed by kernel-2.6.32-220.23.1.el6.x86_64
        zlib-devel is needed by kernel-2.6.32-220.23.1.el6.x86_64
        binutils-devel is needed by kernel-2.6.32-220.23.1.el6.x86_64
        newt-devel is needed by kernel-2.6.32-220.23.1.el6.x86_64
        python-devel is needed by kernel-2.6.32-220.23.1.el6.x86_64
        hmaccalc is needed by kernel-2.6.32-220.23.1.el6.x86_64
 
    # yum install -y gcc redhat-rpm-config patchutils xmlto asciidoc elfutils-libelf-devel zlib-devel binutils-devel newt-devel python-devel hmaccalc
    (其中asciidoc，newt-devel包，在rhel6.5镜像源里没有。需要自行下载，我上传到github上。)
    # yum install -y newt-devel-0.52.11-3.el6.x86_64.rpm slang-devel-2.2.1-1.el6.x86_64.rpm asciidoc-8.4.5-4.1.el6.noarch.rpm
    # yum install -y rng-tools-2-13.el6_2.x86_64    ##安装生成随机数池的rngd软件包
    # rngd -r /dev/urandom      ##生成随机数池
    # SPECS]# rpmbuild -bp kernel.spec      
    # ll rpmbuild/BUILD/        ##做到这里说明，rpm-build已经成功了。
        total 4
        drwxr-xr-x 4 root root 4096 Dec 10 20:45 kernel-2.6.32-220.23.1.el6
    # tar zxf Linux-2.6.32-220.23.1.el6.x86_64.lvs.src.tar.gz   ##解压linux-2.6.32-200.23.1.el6的内核源码包
    # tar zxf Lvs-fullnat-synproxy.tar.gz   ##解压支持FullNAT的lvs包
    # cd lvs-fullnat-synproxy/
    # cp lvs-2.6.32-220.23.1.el6.patch ../rpmbuild/BUILD/kernel-2.6.32-220.23.1.el6/linux-2.6.32-220.23.1.el6.x86_64/
    ##将补丁拷贝到内核文件中
    # cd ../rpmbuild/BUILD/kernel-2.6.32-220.23.1.el6/linux-2.6.32-220.23.1.el6.x86_64/
    # patch -p1 < lvs-2.6.32-220.23.1.el6.patch     ##打补丁
    # vim Makefile
    ...
    EXTRAVERSION =  -220.23.1.el6
    ...
    # make
    # make modules_install
    # make install      ##做到这里，已经编译好了新的内核，开机时候选择新内核。
    # vim /boot/grub/grub.conf  ##修改启动默认内核选项
    ...
    default=0       ##0为第一个选项
    ...

####2.lvs安装:

    # cd lvs-fullnat-synproxy/tools/keepalived/
    # yum install -y popt-devel
    # ./configure --with-kernel-dir="/lib/modules/`uname -r`/build"
    # make
    # make install  ##默认安装在了/usr/local/
    # cd lvs-fullnat-synproxy/tools/ipvsadm/
    # make
    # make install
    # ipvsadm --help    ##多了--FullNAT选项

**至此，已经全部安装好了**
