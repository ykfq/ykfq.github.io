---
title: "CentOS 7.x 系统初始化配置"
excerpt: CentOS 7.x 系统初始化配置，适用于常规用途或用于制作 vmware/openstack 的模版镜像。
permalink: /systems/centos7-system-init
last_modified_at: 2018-08-05
layout: single
toc: true
#header:
#  image: /assets/images/openstack.jpg
#author_profile: true

tags:
  - centos 7.x
  - init

---

This is an system init installation of centos 7 for template use.

### 1. Personalization

```bash
cat >> /root/.bashrc << EOF
HISTTIMEFORMAT="%F %T "
alias ls='ls --color=auto --time-style +"%T %F"'
alias ll='ls -lh'
alias vi='vim'
alias grep='grep --color'

function mkcd
{
  mkdir -p -- "\$@" && cd "\$@"
}
EOF
```

### 2. Set DNS

```bash
sed -i '/^PEERDNS/d' /etc/sysconfig/network-scripts/ifcfg-eth0
sed -e '/^ONBOOT/a\PEERDNS=no' -i /etc/sysconfig/network-scripts/ifcfg-eth0

cat /dev/null > /etc/resolv.conf
cat >> /etc/resolv.conf << EOF
nameserver 119.29.29.29
nameserver 223.5.5.5
EOF
```

### 3. Set timezone/selinux/sshd

```bash
echo "Asia/shanghai" > /etc/timezone && \
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

sed -i '/SELINUX/ s/enforcing/permissive/g' /etc/selinux/config && setenforce 0
sed -i 's/^#UseDNS yes/UseDNS no/' /etc/ssh/sshd_config
sed -i 's/^#PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
```

### 4. Change YUM repositories

```bash
rm -f /etc/yum.repos.d/*
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo
sed -e 's!gpgkey=http://mirrors.aliyun.com/centos!gpgkey=file:///etc/pki/rpm-gpg!g' \
    -e 's!http!https!g' \
    -e 's!mirrors.aliyun.com!mirrors.ustc.edu.cn!g' \
    -i /etc/yum.repos.d/CentOS-Base.repo

yum install -y epel-release
sed -e 's!^mirrorlist=!#mirrorlist=!g' \
    -e 's!^#baseurl=!baseurl=!g' \
    -e 's!//download\.fedoraproject\.org/pub!//mirrors.ustc.edu.cn!g' \
    -e 's!http://mirrors\.ustc!https://mirrors.ustc!g' \
    -i /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel-testing.repo

sed -e 's!enabled=1!enabled=0!g' -i /etc/yum/pluginconf.d/fastestmirror.conf
```

### 5. Install basic tools and set time server

```bash
yum -y install bash-completion bash-completion-extras net-tools telnet \
               mlocate tcpdump gzip unzip bind-utils htop iftop iotop vim \
               rsync lsof wget tree chrony

sed -e 's!centos.pool.ntp.org!cn.pool.ntp.org!g' -i /etc/chrony.conf
systemctl enable chronyd
systemctl restart chronyd
```

### 6. Turn off swap/firewalld if needed
```bash
# If needed
#swapoff /dev/mapper/centos-swap"
#sed -i '/swap/d' /etc/fstab"

systemctl stop firewalld
systemctl disable firewalld
```

### 7. System tunning

Coming soon...
```bash
# Turn off hugepage 
echo never >> /sys/kernel/mm/transparent_hugepage/enabled
echo never >> /sys/kernel/mm/transparent_hugepage/defrag

#  Setting sysctl
cat >> /etc/sysctl.conf << EOF

net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.tcp_max_syn_backlog=65536
net.core.netdev_max_backlog=32768
net.core.somaxconn=32768
net.core.wmem_default=8388608
net.core.rmem_default=8388608
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_timestamps=1
net.ipv4.tcp_synack_retries=2
net.ipv4.tcp_syn_retries=2
net.ipv4.tcp_tw_recycle=0
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_mem=94500000 915000000 927000000
net.ipv4.tcp_max_orphans=3276800
net.ipv4.tcp_fin_timeout=30
net.ipv4.ip_local_port_range=1024 65535
fs.nr_open=10000000
fs.file-max=10000000
kernel.sem=50100 128256000 50100 2560
kernel.shmmax=68719476736
kernel.shmall=68719476736
vm.overcommit_memory=1
vm.swappiness=1
EOF

# Set limits
cat >> /etc/security/limits.d/20-nproc.conf << EOF

*       soft    nproc   100000
root    soft    nproc   unlimited
*       soft    nofile  100000
*       hard    nofile  1000000
*       hard    nproc   1000000
EOF

# Make sysctl take effect
sysctl -p
```
