---
title: "制作 OpenStack 的 CentOS6.x 镜像"
excerpt: 创建 OpenStack 实例时，我们多数时间是使用 镜像+实例类型 来创建的。那么，制作一个通用的镜像就很重要了。
permalink: /openstack/create-centos6-image-for-openstack
last_modified_at: 2017-09-11
layout: category
toc: true
#header:
#  image: /assets/images/openstack.jpg
author_profile: true
---

### 简介
创建 OpenStack 实例时，我们多数时间是使用 镜像+实例类型 来创建的。那么，制作一个通用的镜像就很重要了。

当用户创建一个实例时，会选择相应的镜像（如centos6.8 或者centos7.3 ），再选择相应的实例类型，如`t2.micro`、`t2.large`, 实际他们对应的可能是`2C/2G/20G`或`4C/4G/40G`的硬件配置。

因此，对于我们需要使用的镜像，至少要求提供如下功能：

- 1、合适的操作系统；
- 2、自动扩容硬件配置；
- 3、控制台输出日志；
- 4、带有常用工具；

下面就是`CentOS6.8` 的镜像制作过程。主流镜像制作请参考官网: 
[create-images-manually](https://docs.openstack.org/image-guide/create-images-manually.html)

### 基础环境准备
**在控制节点上执行**
```bash
#安装libvirt相关工具
yum groupinstall Virtualization "Virtualization Client"
yum -y install libvirt

#启动服务
systemctl enable libvirtd; systemctl start libvirtd; systemctl status libvirtd

#下载或从本地上传系统镜像
mkdir /openstack-image
cd /openstack-image
wget http://mirrors.timacloud.cn/centos/6.8/images/CentOS-6.8-x86_64-minimal.iso
```

### 制作镜像
```bash
#创建`qcow2`格式的镜像文件
chown -R qemu:qemu /openstack-image
qemu-img create -f qcow2 /openstack-image/centos6.8-prod.qcow2 10G

#通过virt-install创建虚拟机
virt-install --virt-type kvm --name centos-6.8-mini \
  --ram 2048 --disk ./centos6.8-prod.qcow2,format=qcow2 \
  --network network=default --graphics vnc,listen=0.0.0.0 \
  --noautoconsole --os-type=linux --os-variant=rhel6 \
  --location=./CentOS-6.8-x86_64-minimal.iso
```

### 安装操作系统
安装操作系统，需要通过vnc工具链接到虚拟机终端，在终端内操作。

```bash
#查看镜像vnc监听的端口
netstat -ntlp | grep qemu-kvm
```

查找到虚拟机的vnc端口为`5900`，使用`tigerVNC`进行连接，并在控制台完成系统安装。  
ip就是服务器的ip，端口默认第一个为5900以此类推，也可以通过命令：`virsh vncdisplay vmname`查询端口，推荐使用tigervnc来打开。

![](https://ws1.sinaimg.cn/large/6ee6fa3egy1fqxy9n7ptmj20co059glk.jpg)

以这个方式安装操作系统和正常的安装几乎一样,但是有两点需要注意的:
- 网络: 确保你的网卡eth0是DHCP状态的，而且请务必勾上`auto connect`的对勾。

- 分区: 分区的时候选择默认的`Use all available space`，分区成lvm格式，其他不要设置。

系统安装完毕之后,我们刚才使用的`vnc-install`命令会自动退出。

### 配置虚拟机

在控制节点上使用命令启动虚拟机。
```bash
#查看所有的虚拟机
virsh list --all

#启动虚拟机
virsh start centos6.8
 
#使用tcpump来抓包发现IP地址
tcpdump -i any 'udp port 67 or port 68'

#远程登录主机
ssh 192.168.122.11

#修改虚拟机网络配置
cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << EOF
NAME=eth0
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=dhcp
IPV6INIT=no
PEERDNS=no
EOF

#虚拟机常用设置
#设置bash
cat >> ~/.bashrc << EOF
HISTTIMEFORMAT="%F %T "
alias ls='ls --color=auto --time-style +"%T %F"'
alias ll='ls -lrth'
alias vi='vim'
EOF

#设置公共dns
cat > /etc/resolv.conf << EOF
nameserver 223.5.5.5
nameserver 114.114.114.114
EOF

#设置时钟同步
mkdir -pv /var/log/ntpdate
cat >> /etc/crontab << EOF
#每天晚上2:00 定时同步时钟
00 02 * * * root (/usr/sbin/ntpdate ntp1.aliyum.com) >> /var/log/ntpdate/ntpdate_\$(date +\%Y\%m) 2>&1
EOF

#设置vim
cat >> ~/.vimrc << EOF
colorscheme elflord
"highlight search
set hlsearch
"ignorecase
set ignorecase
"Smart Case, work with ignorecase
set smartcase
"Increase search, dynamic search
set incsearch
"Tab size=4 space
set ts=4
EOF

# 设置Selinux为permissive
sed -i '/SELINUX/ s/enforcing/permissive/g' /etc/selinux/config

#设置ssh连接时禁止dns查询
sed -i 's/^#UseDNS yes/UseDNS no/' /etc/ssh/sshd_config
sed -i '/PasswordAuthentication s/no/yes/g' /etc/ssh/sshd_config

#设置yum
rm -f /etc/yum.repos.d/* && \
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo && \
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo

#安装常用工具
yum -y install bash-completion bash-completion-extras net-tools telnet \
               mlocate tcpdump gzip unzip bind-utils htop iftop iotop vim \
               rsync lsof wget tree ntpdate

#手动同步一次时钟
ntpdate ntp1.aliyum.com

#删除mac设备配置文件
echo "#" > /etc/udev/rules.d/75-persistent-net-generator.rules
#或者删除已生成的配置文件，重启后会重新生成
rm -f /etc/udev/rules.d/70-persistent-net.rules

#增加如下配置已避免连接实例时出现metadata服务问题
cat >> /etc/sysconfig/network << EOF
NOZERCONF=yes
EOF

#关闭防火墙
service iptables stop && chkconfig iptables off
service ip6tables stop && chkconfig ip6tables off

#安装ACPI服务，能让宿主机对虚拟机进行开关机等电源管理操作
yum -y install acpid
chkconfig acpid on

#安装cloud-init, 使得虚拟机能够自动获取密钥设置主机名等
yum install -y cloud-utils-growpart cloud-init cloud-utils parted
 
#修改cloud_init配置文件,增加dns配置引用，在 cloud_init_modules 下面增加:
vim /etc/cloud/cloud.cfg
cloud_init_modules
 - resolv-conf

#去除cloud-init默认阻止root远程登录并且禁止password认证
vim /etc/cloud/cloud.cfg
users：
– defaults
   disable_root：0  # default 1
   ssh_pwauth：  1  # default 0

#安装linux rootfs resize, 使得实例启动时可以自动扩展根分区
yum -y install git
git clone https://github.com/flegmatik/linux-rootfs-resize.git
cd linux-rootfs-resize
./install
cd ../ && rm -rf linux-rootfs-resize

#开启nova console log日志输出支持，centos6和 centos7有差异
#centos6
vim /etc/grub.conf
console=tty0 console=ttyS0,115200n8 #追加到kernel行末尾

#注：/etc/grub.conf /boot/grub/menu.lst 都是指向 /boot/grub/grub.conf 的软链。

#centos7
vim /etc/default/grub
删除rhgb quiet 并追加 console=tty0 console=ttyS0,115200n8

#重新生成grub.cfg文件
grub2-mkconfig -o /boot/grub2/grub.cfg

#清除yum缓存，然后关机
yum clean all
poweroff
```

### 清理和压缩镜像
**在宿主机上操作**
```bash
#清除网络相关硬件生成信息
sudo yum install /usr/bin/virt-sysprep
virt-sysprep -d centos-6.8-mini

#压缩镜像
virt-sparsify --compress ./centos.qcow2 CentOS-6.8-mini-Cloud.qcow2
```
### 添加镜像到glance
```bash
openstack image create "CentOS-6.8-mini-Cloud" \
  --file ./CentOS-6.8-mini-Cloud.qcow2 \
  --disk-format qcow2 \
  --container-format bare --public 
```

### 从virsh删除image模版
确保你制作的镜像已经可用。并且意味着你下次需要重新创建并安装系统。
```bash
virsh list --all
virsh undefine centos6.8
```

