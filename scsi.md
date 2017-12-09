# RHCS集群上用scsi实现文件共享

## 实验环境：*iptable and selinux disabled*
*在RHCS配置上，再增加一台提供磁盘的服务器（172.25.4.3）*

### 开始配置：（只在172.25.4.3）

一.服务端:

    1.首先添加一块磁盘：（通过virt-manager管理器上添加）

    2.软件安装：

    # yum install -y scsi-target-utils-1.0.24-10.el6.x86_64
    # vim /etc/tgt/targets.conf
        
    ...
    # Sample target with one LUN only. Defaults to allow access for all initiators:
        <target iqn.2017-05.com.example:server.target1>
        backing-store /dev/vdb          #指定那块盘
        initiator-address 172.25.4.1    #指定那个IP可以登录使用 
        initiator-address 172.25.4.2        
        </target>
    ...

    # /etc/init.d/tgtd start   ##开启服务
    # tgt-admin -s             ##查看是否配置成功

二.客户端：

    # yum install -y iscsi-initiator-utils
    # iscsiadm -m discovery -t st -p 172.25.66.3  #1.发现设备，开机自动开启
    # chkconfig --list iscsid             #查看是否开机自启
    # chkconfig --list iscsi              #查看是否开机自启
    # iscsiadm -m node -l                 #2.登陆
    # /var/lib/iscsi/nodes                #有登陆设备的信息
    # cat /proc/partitions                #查看分区信息，sda是存在
    # fdisk -cu /dev/sda                  #分区,n新建，d删除，w保存,
                                          新的分区格式化成(mkfs.ext4 
                                          /dev/sda1 )后可以挂载。
**利用集群进行挂载，添加集群资源data，如图:**

![](http://mdpicture-1253499256.file.myqcloud.com/Screenshot%20from%202017-12-04%2020-48-25.png)

**用clustat命令查看集群状态,如图:**

![](http://mdpicture-1253499256.file.myqcloud.com/Screenshot%20from%202017-12-04%2020-49-50.png)

**然后，重新启动apahce:**

    # clusvcadm -e apache               ##启动apache资源组
    # clusvcadm -d apache               ##停止apache资源组
    # clusvcadm -e apache -m server1    ##启动apache资源组，到指定server1服务器
    # clusvcadm -e apache -m server2    ##将apache资源组回迁到server2  

**那么问题来了，ext4是本地文件系统，不能实现存储共享。如何解决呢？**

*方案：*

    可试用nfs文件系统，gfs2文件系统，gfs2是常用集群文件系统。
    直接格式化成gfs2后，挂载，直接启动资源会报错，需要到页面重新设置，选择Autodetect类型。

    # mkfs.gfs2 -p lock_dlm -t westosha:mygfs2 -j 3 /dev/sda1   ##选择Y

    通过blkid查看/dev/sda1的UUID，并且修改，/etc/fstab
    # vim /etc/fstab
    ...
    UUID=1742c3ae-04fb-6ed3-84d8-330939833f23       /var/www/html   gfs2 _netdev 0 0
    ...

    同样，在另外一台机器也需要修改/etc/fstab。

    测试：
    启动apache资源组，查看挂载情况。
    另一台机器上执行：
    # mount -a
    然后修改主页内容，刷新访问主页，内容也改变，做到这里可以判断，存储已经共享了。

    /boot 不可以用lvm分区，因为lvm驱动在内核加载，而内核在mbr指向/boot后再加载的，所以/boot不可以用lvm。

**那么问题又来了，ext4不可延伸,如何解决？**

*方案:*

用lvm实现延伸：

    # fdisk -cu /dev/sda
    n-->p-->空格-->+2G-->w       ##分出一块新的分区
    # cat /proc/partitions      ##查看分区信息
    # partprobe                 ##刷新

    # pvcreate /dev/sda1        ##创建pv
    # pvs
    PV         VG   Fmt  Attr PSize PFree
    /dev/sda1       lvm2 a--  4.00g 4.00g
    # vgcreate cluster_vg /dev/sda1     ##创建vg
    Clustered volume group "cluster_vg" successfully created
    # vgs
    VG         #PV #LV #SN Attr   VSize VFree
    cluster_vg   1   0   0 wz--nc 4.00g 4.00g
    # lvcreate -n demo -L 4G cluster_vg ##创建lv,错误，直接执行下面的。
    Volume group "cluster_vg" has insufficient free space (1023 extents): 1024 required.
    # lvcreate -n demo -l 1023 cluster_vg
    Logical volume "demo" created
    lvs
    LV   VG         Attr       LSize Pool Origin Data%  Move Log Cpy%Sync Convert demo cluster_vg -wi-a----- 4.00g

    # mkfs.gfs2 -p lock_dlm -t westosha:mygfs2 -j 3 /dev/cluster_vg/demo            ##格式化为gfs2

    通过blkid查看/dev/sda1的UUID，并且修改，/etc/fstab
    # vim /etc/fstab
    ...
    UUID=1742c3ae-04fb-6ed3-84d8-330939833f23(根据实际填写)       /var/www/html       gfs2 _netdev0 0
    ...
    # mount -a  ##自动挂载
    扩充：
    在添加一个分区：
    # fdisk -cu /dev/sda
    n-->p-->2-->空格-->+2G-->w          ##分出一块新的分区
    # pvcreate /dev/sda2 
    Physical volume "/dev/sda2" successfully created
    # pvs
    PV         VG         Fmt  Attr PSize PFree
    /dev/sda1  cluster_vg lvm2 a--  4.00g    0 
    /dev/sda2  cluster_vg lvm2 a--  4.00g 4.00g
    [root@server1 ~]# vgextend cluster_vg /dev/sda2 
    Volume group "cluster_vg" successfully extended
    # vgs
    VG         #PV #LV #SN Attr   VSize VFree
    cluster_vg   2   1   0 wz--nc 7.99g 4.00g
    # lvextend -l +1023 /dev/cluster_vg/demo 
    Extending logical volume demo to 7.99 GiB
    Logical volume demo successfully resized
    # lvs
    LV   VG         Attr       LSize Pool Origin Data%  Move Log Cpy%Sync Convert
    demo cluster_vg -wi-ao---- 7.99g

**执行:**

    # gfs2_grow /dev/cluster_vg/demo    ##刷新容量

    最后删除步骤：
    # clusvcadm -d apache       ##关闭apache资源组
    Local machine disabling service:apache...Success
    # vim /etc/fstab
    将之前添加的删除掉，两台都要执行。

    # lvremove /dev/cluster_vg/demo     ##删除lv
    Do you really want to remove active clustered logical volume demo? [y/n]: y
    Logical volume "demo" successfully removed
    # vgremove cluster_vg           ##删除vg
    Volume group "cluster_vg" successfully removed
    # pvremove /dev/sda1            ##从pv拔除sda1
    Labels on physical volume "/dev/sda1" successfully wiped
    # pvremove /dev/sda2            ##从pv上拔除sda2
    Labels on physical volume "/dev/sda2" successfully wiped

    fdisk -cu /dev/sda
    删除掉sda1,sda2

    最后在，集群页面中，移除apache资源组的data，并删除data资源。
