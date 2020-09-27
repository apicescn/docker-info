# Centos7服务器基本配置

<a name="2d6ysi"></a>
# 1 修改主机名称
```bash
hostnamectl --static set-hostname  k8s-master01
```


<a name="a0i5wo"></a>
# 2 关闭防火墙和SELINUX
```bash
systemctl disable firewalld
systemctl stop firewalld
sed -i 's/SELINUX=permissive/SELINUX=disabled/g' /etc/selinux/config
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```


<a name="CYLZZ"></a>
# 3 设置IPV4转发
1）CentOS7 下可编辑配置文件：
```bash
vi /etc/sysctl.conf
```

<br />2）设置：
```bash
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
或者
```bash
cat >> /etc/sysctl.conf<<EOF
net.ipv4.ip_forward=1
watchdog_thresh=30
net.bridge.bridge-nf-call-iptables=1
net.ipv4.neigh.default.gc_thresh1=4096
net.ipv4.neigh.default.gc_thresh2=6144
net.ipv4.neigh.default.gc_thresh3=8192
EOF
```
3）执行如下命令生效：
```bash
#sudo sysctl -p
```

<br />

<a name="PPwox"></a>
# 4 禁用swap
Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。可以通过kubelet的启动参数–fail-swap-on=false更改这个限制。    <br />修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m确认swap已经关闭。    <br />`swapoff -a` 命令只是临时关闭swap，重启后有复原。
```bash
#vi /etc/fstab

#
# /etc/fstab
# Created by anaconda on Fri Jan 12 13:47:45 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=d92230b6-422f-4a73-8a7e-8db773d22d2a /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```
将最后一行注释掉。<br />

<a name="kHD7V"></a>
# 5 启用cgroup
修改配置文件/etc/default/grub，启用cgroup内存限额功能,配置两个参数：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory swapaccount=1"
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
```
注意：要执行sudo update-grub 或（grub2-mkconfig -o /boot/grub2/grub.cfg）更新grub，然后重启系统后生效。

<a name="lg9rhm"></a>
# 6 挂载磁盘
阿里云数据盘的设备名由系统默认分配，从 /dev/xvdb 开始往后顺序排列，分布范围包括 /dev/xvdb−/dev/xvdz。
<a name="unccpq"></a>
## 6.1 查看磁盘
远程连接实例。运行**`fdisk -l` **命令查看实例是否有数据盘。如果执行命令后，没有发现 /dev/vdb，表示您的<br />实例没有数据盘，无需格式化数据盘，请忽略本文后续内容。<br />如果您的数据盘显示的是 dev/xvd?，表示您使用的是非 I/O 优化实例。其中 ? 是 a−z 的任一个字母。<br />
<br />查看磁盘：
```bash
# fdisk -l

Disk /dev/vda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0008de3e

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    83884031    41940992   83  Linux

Disk /dev/vdb: 751.6 GB, 751619276800 bytes, 1468006400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
从上内容可以发现/dev/vdb磁盘有750G,设备启动列表中只有/dev/vda1，没有/dev/vdb1,说明/dev/vdb没有挂载。如果/dev/vdb已经挂载，就无需下面的挂载操作。<br />

<a name="7hmgzg"></a>
## 6.2 创建一个单分区数据盘
创建一个单分区数据盘，依次执行以下命令：

1. 运行 **`fdisk -u /dev/vdb`**：对数据盘进行分区。<br />
1. 输入 n 并按回车键：创建一个新分区。<br />
1. 输入 p 并按回车键：选择主分区。因为创建的是一个单分区数据盘，所以只需要创建主分区。如果要创建 4 个以上的分区，您应该创建至少一个扩展分区，即选择 e。<br />
1. 输入分区编号并按回车键。因为这里仅创建一个分区，可以输入 1。<br />
1. 输入第一个可用的扇区编号：按回车键采用默认值 1。<br />
1. 输入最后一个扇区编号：因为这里仅创建一个分区，所以按回车键采用默认值。<br />
1. 输入 wq 并按回车键，开始分区。<br />



```bash
[root@iXXXXXXX ~]# fdisk -u /dev/vdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0x5f46a8a2.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.
Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)
WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
switch off the mode (command 'c') and change display units to
sectors (command 'u').
Command (m for help): n
Command action
e extended
p primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-41610, default 1): 1
Last cylinder, +cylinders or +size{K,M,G} (1-41610, default 41610):
Using default value 41610
Command (m for help): wq
The partition table has been altered!
Calling ioctl() to re-read partition table.
Syncing disks.
```


<a name="fw2xhv"></a>
## 6.3 查看新的分区

<br />运行命令 **fdisk -l**。如果出现以下信息，说明已经成功创建了新分区 /dev/vdb1。
```bash
[root@iXXXXXXX ~]# fdisk -l
Disk /dev/vda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00053156
Device Boot Start End Blocks Id System
/dev/vda1 * 1 5222 41942016 83 Linux
Disk /dev/vdb: 21.5 GB, 21474836480 bytes
16 heads, 63 sectors/track, 41610 cylinders
Units = cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x5f46a8a2
Device Boot Start End Blocks Id System
/dev/vdb1 1 41610 20971408+ 83 Linux
```


<a name="s6bvsl"></a>
## 6.4 格式化分区
在新分区上创建一个文件系统：运行命令 **mkfs.ext4 /dev/vdb1**。<br />本示例要创建一个 ext4 文件系统。您也可以根据自己的需要，选择创建其他文件系统，例如，如果需要在 Linux、Windows 和 Mac 系统之间共享文件，您可以使用 mkfs.vfat 创建 VFAT 文件系统。<br />创建文件系统所需时间取决于数据盘大小。
```bash
[root@iXXXXXXX ~]# mkfs.ext4 /dev/vdb1
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
32768000 inodes, 131071744 blocks
6553587 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2279604224
4000 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```


<a name="uofuye"></a>
## 6.5 向 /etc/fstab 写入新分区信息
建议备份 etc/fstab：运行命令 **cp /etc/fstab /etc/fstab.bak**。<br />向 /etc/fstab 写入新分区信息：运行命令 **echo /dev/vdb1 /mnt ext4 defaults 0 0 >> /etc/fstab**。<br />
<br />Ubuntu 12.04 不支持 barrier，所以对该系统正确的命令是：echo '/dev/vdb1 /mnt ext4 barrier=0 0 0' >> /etc/fstab。<br />如果需要把数据盘单独挂载到某个文件夹，比如单独用来存放网页，请将以上命令 /mnt 替换成所需的挂载点路径。<br />

<a name="limgwg"></a>
## 6.6 查看 /etc/fstab 中的新分区信息
查看 /etc/fstab 中的新分区信息：运行命令 **`cat /etc/fstab`**。
```bash
[root@iXXXXXXX ~]# cat /etc/fstab
#
# /etc/fstab
# Created by anaconda on Thu Feb 23 07:28:22 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=3d083579-f5d9-4df5-9347-8d27925805d4 / ext4 defaults 1 1
tmpfs /dev/shm tmpfs defaults 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
sysfs /sys sysfs defaults 0 0
proc /proc proc defaults 0 0
/dev/vdb1 /mnt ext4 defaults 0 0
```


<a name="ees9gb"></a>
## 6.7 挂载文件系统
挂载文件系统：运行命令 **mount /dev/vdb1 /mnt**。<br />查看目前磁盘空间和使用情况：运行命令 **df -h**。如果出现新建文件系统的信息，说明挂载成功，可以使用新的文件系统了。<br />挂载操作完成后，不需要重启实例即可开始使用新的文件系统。
```bash
[root@iXXXXXXX ~]# mount /dev/vdb1 /mnt
[root@iXXXXXXX ~]# df -h
Filesystem Size Used Avail Use% Mounted on
/dev/vda1 40G 6.6G 31G 18% /
tmpfs 499M 0 499M 0% /dev/shm
/dev/vdb1 20G 173M 19G 1% /mnt
```

<br />其他操作请参考文档完成磁盘挂载：[https://help.aliyun.com/document_detail/25426.html?spm=5176.7738005.2.3.VwfXvE](https://help.aliyun.com/document_detail/25426.html?spm=5176.7738005.2.3.VwfXvE)<br />
<br />**重要提示：严格按照文档提示要求做。并且挂载完成后，请重启虚机，看系统是否正常启动，并且使用命令 df -h 或者 fdisk -l 数据盘正常挂载。**<br />
<br />

