# Centos7安装docker

<a name="kmxcwg"></a>
# 1  存储驱动类型  
选择overlay2，overlay2已经作为docker CE的默认存储驱动类型，centos系统使用overlay2需要ext4或xfs文件系统支持。<br />docker存储驱动介绍：<br />官方：[https://docs.docker.com/storage/storagedriver/select-storage-driver/#docker-for-mac-and-docker-for-windows](https://docs.docker.com/storage/storagedriver/select-storage-driver/#docker-for-mac-and-docker-for-windows)<br />中文：[http://dockone.io/article/1765](http://dockone.io/article/1765)<br /> 
<a name="z6ydug"></a>
# 2  升级内核
运行docker的node节点需要升级到4.x/5.x内核以上才支持overlay2驱动
```bash
#检查内核，如果非4.n/5.n开头需升级内核 
uname -sr

#添加升级内核的第三方库
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm

#列出内核相关包
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

#安装最新稳定版
yum --enablerepo=elrepo-kernel install kernel-ml -y

#查看内核默认启动顺序
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
#结果显示，按顺序index 为0，1，2，3
CentOS Linux (4.18.5-1.el7.elrepo.x86_64) 7 (Core)                 --0
CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)                  --1
CentOS Linux (3.10.0-514.el7.x86_64) 7 (Core)                      --2
CentOS Linux (0-rescue-606a1594a76c46ba8397ba7f3a0cd90c) 7 (Core)  --3
或
CentOS Linux 7 Rescue 987faebec506464aabfe9c8b44b59e60 (5.2.9-1.el7.elrepo.x86_64)
CentOS Linux (5.2.9-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (3.10.0-514.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-95a9f7b6c7764c1c9d7f9faa3a7af807) 7 (Core)

#设置默认启动的内核，每个人机器不一样，看清楚选择自己的index对应是否为新安装的内核
grub2-set-default 0

#重启才生效
reboot

#检查内核,4.n/5.n即升级成功
uname -a
```
参考：[https://imroc.io/posts/kubernetes/install-kubernetes-1.9-on-centos7-with-kubeadm/](https://imroc.io/posts/kubernetes/install-kubernetes-1.9-on-centos7-with-kubeadm/)
<a name="xg1quv"></a>
# 3  安装docker
<a name="vgsimf"></a>
## 3.1  安装docker
```bash
#添加docker阿里提供的CentOS 7的yum源
cat >> /etc/yum.repos.d/docker.repo <<EOF
[docker-repo]
name=Docker Repository
baseurl=http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
enabled=1
gpgcheck=0
EOF

#查看docker版本信息，指定docker安装版本
#yum list docker-ce --showduplicates | sort -r

#使用ovlerlay2,解决 ovlerlay2 兼容性问题，要确保 yum-plugin-ovl安装
yum install -y yum-plugin-ovl

#安装docker
yum install -y docker-ce-18.09.0-3.el7.x86_64 docker-ce-cli-18.09.0-3.el7.x86_64 containerd.io
```

如上述安装yum-plugin-ovl报aliyun服务器**404问题**时，可将yum源更改如下即可。

```bash
#添加docker官方提供的CentOS7的yum源
cat >> /etc/yum.repos.d/docker.repo <<EOF
[docker-repo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7
enabled=1
gpgcheck=0
EOF
```
由于docker将于3月31日关闭dockerproject的托管，故而上术的yum地址需更换为：<br />[https://download.docker.com/linux/centos/7](https://download.docker.com/linux/centos/7/)<br />即执行如下命令：

```bash
#添加docker官方提供的CentOS7的yum源
cat >> /etc/yum.repos.d/docker.repo <<EOF
[docker-repo]
name=Docker Repository
baseurl=https://download.docker.com/linux/centos/7/x86_64/stable
enabled=1
gpgcheck=0
EOF
```

如果安装时无法找到对应的安装包时，可执行如下命令：

```bash
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
注：此处选择18.09.0版本是因为Kubernetes官方说明文档中有说明验证过的docker版本分别为:17.03.x,18.06.x,18.09.x，故而本次安装使用18.09.0版本，具体参见：[https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.15.md#unchanged](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.15.md#unchanged)

<a name="roinmk"></a>
## 3.2  配置docker
```bash
#新增文件夹
mkdir -p /etc/docker

#写入配置
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors":["https://pxz0rob3.mirror.aliyuncs.com"],
  "insecure-registries":["172.16.1.0/24"],
  "graph":"/mnt/docker",
  "storage-driver":"overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "bip":"10.10.10.0/22",
  "log-driver":"json-file",
  "log-opts":{
    "max-size":"100m",
    "max-file":"20"
  }
}
EOF
```
registry-mirrors：配置安全的镜像服务，链接必须是安全的https<br />insecure-registries：配置不安全的镜像服务<br />graph：配置docker存储位置，运行时根目录<br />storage-driver：存储类型，overlay2需要升级内核<br />storege-opts: 如果使用 Docker EE 并且版本大于 17.06，需要配置这个<br />bip：docker内部网络地址段<br />log-driver：配置驱动类型，默认为json-file<br />log-opt: 日志配置，配置日志文件大小和保留日志文件个数

重点是需要配置**log-driver**和**log-opts**。<br />配置参考如下：https://docs.docker.com/engine/reference/commandline/dockerd/

<a name="74r5dd"></a>
## 3.3 启动docker
```bash
#刷新配置,使修改的配置生效
systemctl daemon-reload

#设置开机启动
systemctl enable docker

#启动docker
systemctl start docker
```

<a name="cxidsm"></a>
# 4 卸载docker
```bash
#查询已经安装docker包
$ yum list installed | grep docker

docker-ce.x86_64                    18.09.0.ce-3.el7           @docker-ce-stable

#删除docker包
$ sudo yum -y remove docker-ce.x86_64

#删除工作目录
$ rm -rf /var/lib/docker
```

