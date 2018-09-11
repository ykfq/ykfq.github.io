---
title: "如何在 CentOS 6.x 上安装 Java7 Tomcat7 MySQL5.5"
excerpt: "在 CentOS 6.x 上快速安装 Java7、Tomcat7 和 MySQL5.5 服务。"
permalink: /systems/install-java7-tomcat7-mysql55-on-centos6
last_modified_at: 2018-08-13
#layout: single

#header:
#  image: /assets/images/openstack.jpg
#author_profile: true

tags:
  - java
  - mysql
  - tomcat
  - centos 6.x

---

### 需求收集
- 系统环境: `CentOS 6.8`
- 硬件配置: `2C/4G/60G`
- 应用版本:   
  - `Java1.7 最新版`
    > https://mirror.its.sfu.ca/mirror/CentOS-Third-Party/NSG/common/x86_64/jdk-7u80-linux-x64.rpm
  
  - `mysql5.5 最新版`
    > https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-devel-5.5.59-1.el6.x86_64.rpm  
    > https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-server-5.5.59-1.el6.x86_64.rpm  
    > https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-client-5.5.59-1.el6.x86_64.rpm  
  
  - `Tomcat7.0 最新版`
    > https://mirrors.aliyun.com/apache/tomcat/tomcat-7/v7.0.85/bin/apache-tomcat-7.0.85.tar.gz

以上需求不做硬性要求, 如果你了解其它版本的差异和兼容性并能够自行解决, 可以使用其它版本;

### 安装Java1.7
```bash
#rpm 包安装
jdk_version=7u80
rpm -ivh https://mirror.its.sfu.ca/mirror/CentOS-Third-Party/NSG/common/x86_64/jdk-${jdk_version}-linux-x64.rpm --prefix=/usr/local/java

#添加JAVA环境变量
cat >> /etc/profile << EOF
export JAVA_HOME=/usr/local/java/jdk${jdk_version}
export JRE_HOME=/usr/local/java/jdk${jdk_version}/jre
export PATH=\$PATH:\$JAVA_HOME/bin:\$JRE_HOME/bin
EOF

source /etc/profile
java -version
```

### 安装MySQL5.5
```bash
# 查找并卸载旧版mysql
rpm -qa |grep -i mysql | xargs yum -y remove

# 安装依赖
yum -y groupinstall "Development tools"
yum -y install perl-DBI

# 指定mysql版本并使用yum在线安装
MYSQL_VERSION=5.5.61-1
yum -y install https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-devel-${MYSQL_VERSION}.el6.x86_64.rpm
yum -y install https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-server-${MYSQL_VERSION}.el6.x86_64.rpm
yum -y install https://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-5.5/MySQL-client-${MYSQL_VERSION}.el6.x86_64.rpm

# 复制配置文件和启动脚本
cp /usr/share/mysql/my-medium.cnf /etc/my.cnf
cp /usr/share/mysql/mysql.server /etc/init.d/mysqld

# 添加优化参数
sed -i '/\[mysqld]/a\character_set_server=utf8' /etc/my.cnf
sed -i '/\[mysqld]/a\user=mysql' /etc/my.cnf

# 启动服务并设置初始化密码
/etc/init.d/mysqld restart
/usr/bin/mysqladmin -u root -h localhost.localdomain password 'tE&%a5BBVzsPBfmN7JZt'

# 登录数据库
mysql -hlocalhost.localdomain -uroot -p"tE&%a5BBVzsPBfmN7JZt"
```

### 安装Tomcat
- 安装Tomcat7.0
```bash
TOMCAT_VERSION=7.0.81
wget http://mirrors.hust.edu.cn/apache/tomcat/tomcat-7/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz
tar xf apache-tomcat-${TOMCAT_VERSION}.tar.gz
mv apache-tomcat-${TOMCAT_VERSION} /opt && cd /opt && ln -sf apache-tomcat-${TOMCAT_VERSION} tomcat
```
- 安装`JDBC`
```bash
MYSQL_CONNECTOR_VERION=5.1.46
wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-${MYSQL_CONNECTOR_VERION}.tar.gz
tar xf mysql-connector-java-${MYSQL_CONNECTOR_VERION}.tar.gz
cp mysql-connector-java-${MYSQL_CONNECTOR_VERION}/mysql-connector-java-${MYSQL_CONNECTOR_VERION}-bin.jar ${JAVA_HOME}/
ln -sf ${JAVA_HOME}/mysql-connector-java-${MYSQL_CONNECTOR_VERION}-bin.jar ${JAVA_HOME}/mysql-connector-java.jar
```