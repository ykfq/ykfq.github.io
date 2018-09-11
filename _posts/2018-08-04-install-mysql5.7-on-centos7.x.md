---
title: "如何在 CentOS 7.x 上安装 MySQL 5.7"
excerpt: "如何在 CentOS 7.x 上安装 MySQL 5.7"
permalink: /databases/how-to-install-mysql57-on-centos7x
last_modified_at: 2018-07-28
layout: tags
#header:
#  image: /assets/images/openstack.jpg
#author_profile: true

tags:
  - secret
  - configMap
  - binary

---

### 1. 安装mysql5.7

#### CentOS6.x
 使用中国科技大学镜像源：https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/ 
```bash
version=5.7.22-1  
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-libs-${version}.el6.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-devel-${version}.el6.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-common-${version}.el6.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-server-${version}.el6.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-client-${version}.el6.x86_64.rpm

yum -y remove mariadb*
yum -y install numactl
yum -y localinstall mysql-community-*.rpm

sed -i '/mysqld/a\user=mysql' /etc/my.cnf
chkconfig mysqld on
service mysqld start 
grep "temporary password" /var/log/mysqld.log
mysql_secure_installation
mysql -uroot -p
```

#### CentOS7.x
 使用中国科技大学镜像源：https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/
 ```bash
version=5.7.22-1  
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-libs-${version}.el7.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-devel-${version}.el7.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-common-${version}.el7.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-server-${version}.el7.x86_64.rpm
curl -SLO https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.7/mysql-community-client-${version}.el7.x86_64.rpm

yum -y remove mariadb*
yum -y install numactl
yum -y localinstall mysql-community-*.rpm

sed -i '/\[mysqld\]/a\performance_schema = ON' /etc/my.cnf
sed -i '/\[mysqld\]/a\user=mysql' /etc/my.cnf
sed -i '/mysql.sock/a\character_set_server = utf8' /etc/my.cnf

systemctl enable mysqld; systemctl start mysqld

#修改密码强度策略(一定要在上面那步初始化启动完成之后再修改)
sed -i '/mysql.sock/a\validate_password_policy = 0' /etc/my.cnf
systemctl restart mysqld

grep "temporary password" /var/log/mysqld.log | awk -F"localhost: " '{print $2}'
mysql_secure_installation
mysql -uroot -p
```

### 2、安装mysql5.5
 Remine 只支持mysql5.0-5.5，mysql>=5.6 有问题，详见官方说明https://www.redmine.org/projects/redmine/wiki/RedmineInstall/
 使用中国科技大学镜像源：https://mirrors.ustc.edu.cn/mysql-ftp/Downloads
 ```bash
 wget https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-5.5.55-1.el6.x86_64.rpm-bundle.tar
 tar xf MySQL*
 rpm -ivh MySQL*.rpm
 chkconfig mysqld on
 service mysqld start
 mysql_secure_installation
```

### 3. 报错问题
#### 1、无法连接到服务器：Table 'performance_schema.session_variables' doesn't exist
```bash
#先在配置文件开启performance_schema
[mysqld]
performance_schema=ON

#再登录mysql更新库表：
mysql_upgrade -u root -p --force
```
https://stackoverflow.com/questions/36746677/table-performance-schema-session-variables-doesnt-exist
https://dev.mysql.com/doc/refman/5.7/en/performance-schema-quick-start.html


#### 2、Fatal error: mysql.user table is damaged. Please run mysql_upgrade

执行时 `mysql_upgrade -u root -p --force` 却提示因`mysqld`没有启动而无法连接到mysql，岂不是进入了死局？

还好stackoverflow 找到了答案:先跳过授权启动mysqld：  
`mysqld --skip-grant-tables`

如果依然提示 
`mysql_upgrade: Got error: 2002: Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock'`

需要修改配置文件，指定mysql的进程位置：
```bash
[client]
socket=/data/mysql/mysql.sock
```

然后新开一个克隆窗口，执行
```bash
mysql_upgrade -u root -p 
```
完成后就可以了正常启动mysqld 了，最后`pkill mysqld` ，使用正常方式启动：
```bash
service mysqld start 
```

> https://stackoverflow.com/questions/36156475/mysqld-doesnt-start-after-brew-upgrade-from-5-6-to-5-7
> https://serverfault.com/questions/527422/mysql-upgrade-is-failing-with-no-real-reason-given

#### 3、1 of ORDER BY clause is not in GROUP BY clause and contains nonaggregated column
=='information_schema.PROFILING.SEQ' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by==

`vim /etc/my.cnf`
```bash
sql_mode=""
```


#### 4、忘记root密码
使用非root用户启动mysqld
在配置文件中添加 
```bash
[mysqld]
user=mysql
```
```bash
update mysql.user set authentication_string=password('123456') where user='root' and Host ='localhost';
```
> http://blog.csdn.net/xinliuqianxue/article/details/52156568 

#### 5、You must reset your password using ALTER USER statement before executing this statement.
```bash
SET PASSWORD = PASSWORD('xxxxxx#345');
ALTER USER root@'localhost' PASSWORD EXPIRE NEVER;
flush privileges;
```
#### 6、[ERROR] Fatal error: Please read "Security" section of the manual to find out how to run mysqld as root!

在my.cnf文件中，指定`user=mysql`;


#### 7、ERROE --initialize specified but the data directory has files in it

每次重启都会提示，数据已经存在了，但是我的进程并没有启动，而且手动删除了 `/var/lib/mysql` 下的数据，这是为什么呢？

`systemctl status mysqld.service` 显示服务启动失败，并且有上面的提示；

`tail -f /var/log/mysqld.log` 发现不断在刷日志；

`ps -ef | grep mysql` 竟然有进程存在，见鬼了。
```
[root@izm5eg3yspb12eu8z0ocfsz ~]# ps -ef | grep mysql
mysql    15669     1  0 11:50 ?        00:00:00 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql    15672     1  0 11:50 ?        00:00:00 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
root     15687  1260  0 11:50 pts/1    00:00:00 grep --color=auto mysql
```


### 参考文档

> http://www.tecmint.com/install-latest-mysql-on-rhel-centos-and-fedora/
> https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-centos-7
