---
title: "3步修改 CentOS7 网卡名为 eth0"
excerpt: 简单3步，将 CentOS7 网卡名由 ensxx 修改为 eth0。
permalink: /systems/change-centos7-ifcfg-name-in-three-steps
last_modified_at: 2018-11-18
layout: single
toc: true
#header:
#  image: /assets/images/openstack.jpg
#author_profile: true

tags:
  - centos 7.x
  - init
  - grub
  - eth0

---

使用简单的3步，将 CentOS7 网卡名由 ensxx 修改为 eth0。

### 1、修改网卡配置文件

- 更改 ifcfg 文件名
  ```bash
  ls /etc/sysconfig/network-scripts/ifcfg-ens* | xargs -I '{}' mv '{}' /etc/sysconfig/network-scripts/ifcfg-eth0
  ```
  其实这一步不是必须的，直接执行下一步创建一个新文件也可以。

- 配置IP地址
  修改 NAME 和 DEVICE 的值为 eth0。可按需配置固定IP。
  ```bash
  cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << EOF
  TYPE=Ethernet
  DEVICE=eth0
  NAME=eth0
  ONBOOT=yes
  BOOTPROTO=dhcp
  PEERDNS=no
  IPV6INIT=no
  EOF
  ```

  这一步完成若，重启网络会失败的，因为网卡和设备名变了。

### 2、修改内核启动参数

- 修改 grub 文件  

  编辑 `/etc/default/grub`, 在`GRUB_CMDLINE_LINUX`这一行末尾引号内添加`net.ifnames=0 biosdevname=0`，如下图：  
  ![image](https://ws1.sinaimg.cn/large/6ee6fa3egy1fnjphna6luj20ri03w3yp.jpg)
  
  ```bash
  # 一条命令实现
  sed -e 's/\<quiet\>/& net.ifnames=0 biosdevname=0/' -i /etc/default/grub
  ```
  以上命令参考: https://www.golinuxhub.com/2017/06/sed-insert-word-after-match-in-middle.html

- 重新生成 GRUB 配置文件
  ```bash
  grub2-mkconfig -o /boot/grub2/grub.cfg
  ```

### 3、禁用 NetworkManager（按需使用）
```bash
 systemctl disable NetworkManager
```

### 4、重启系统
```bash
reboot
```

linux中查看网卡mac地址
```bash
ifconfig -a #其中 HWaddr 字段就是mac地址
cat /sys/class/net/eth0/address #查看eth0的mac地址
cat /proc/net/arp #查看连接到本机的远端ip的mac地址
```

参考
> http://www.centoscn.com/CentOS/config/2015/0404/5094.html  
> https://www.thegeekdiary.com/centos-rhel-7-how-to-modify-network-interface-names/

