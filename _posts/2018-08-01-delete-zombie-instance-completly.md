---
title: "如何从 OpenStack 中彻底删除僵尸实例"
excerpt: 在使用运维 OpenStack 的过程中，我们可能会遇到系统错误引起实例删除失败。通常是 Dashboard 上显示实例已经不存在了，但 MySQL 数据库中还有很多冗余的信息。
permalink: /openstack/delete-zombie-instances-completly
last_modified_at: 2018-08-01
layout: single
#header:
#  image: /assets/images/openstack.jpg
# author_profile: true
tags:
 - openstack
 - zombie
 - mysql

---

### 1、正常删除实例（ocata）
> https://docs.openstack.org/user-guide/cli-delete-an-instance.html

- 1.1 List all instances:

  ```
  openstack server list
  ```

- 1.2 Run the openstack server delete command to delete the instance. The following example shows deletion of the  newServer instance, which is in  ERROR state:
  ```
  openstack server delete newServer
  ```

- 1.3 To verify that the server was deleted, run the openstack server list command:
  ```
  openstack server list
  ```

- 1.4 清理之前"临时关闭外键检查"的遗留问题

  > http://blog.csdn.net/spch2008/article/details/7952369

  - 删除 `security_group_instance_association` 中关联数据:
    ```
    delete from security_group_instance_association where instance_id=xxxxx;
    ```
    我这里没有相关实例，略过。

  - 删除 `instance_info_caches` 中关联数据
    ```bash
    # 先看下表中有哪些字段，长什么样子：
    select * from instance_info_caches \G;
    
    # 筛选出有用的信息：
    select instance_uuid,deleted,network_info from instance_info_caches where network_info="[]";
    
    # 确认network_info 为空的实例已经被删除了，这里一并删除之：
    delete from instance_info_caches where network_info="[]";
    
    # 删除因在web界面删除失败，而我在instances表中直接删除的实例：          
    delete from instance_info_caches where network_info like "%172.20.69.%";
    delete from instance_info_caches where network_info like "%172.20.70.%";
    
    #删除instance镜像文件
    uuids2="
      0a2d1bf7-982c-4034-9713-566d58ced0c4
      ddeb6e86-713e-44b0-b27c-88ae1cdece3f
      596aa6ca-7abe-46ea-b375-9808164b6073
      1ed0a4dd-ddfe-4be0-97af-9e0c20ccdaae
      b3a82ba7-2046-45da-9744-ed935a185edf
    "
    for id in $uuids2;do rm -rf /var/lib/nova/**instances**/$id;done
    ```

### 2、删除僵尸实例

> http://6728496.blog.51cto.com/6718496/1178393

前天强制重启一台 OpenStack Nova 控制节点以后发现虚拟机消失，但是 `nova-list` 命令显示 instances 仍然是 `running` 的状态，使用 `nova-delete` 终止命令仍然无效，暂时把这样的 instance 称作 "僵尸实例（zombie instance）"：
```bash
virsh list
Id Name         State
----------------------------------

euca-describe-instances
#RESERVATION    r-bkl83j20    bangcloud    default
#INSTANCE    i-0000001d    ami-00000002    172.16.39.121    172.16.39.121    running    vpsee (vpseecloud, node00)    0            2011-11-10T12:45:12Z    nova    aki-00000001    ami-00000000
#RESERVATION    r-j335q6ny    bangcloud    default
#INSTANCE    i-0000001e    ami-00000002    172.16.39.122    172.16.39.122    running    vpsee (vpseecloud, node00)    0            2011-11-10T12:54:27Z    nova    aki-00000001    ami-00000000

euca-terminate-instances i-0000001d
euca-terminate-instances i-0000001e

#和 删除 OpenStack Nova Volume 时遇到的 error_deleting 问题 这篇文章提到的解决办法一样，直接操作数据库来删除这2条僵尸实例的记录。登录 mysql，使用 nova 数据库，找出要删除 instance 的 id，然后删除：

mysql -u root -p
#Enter password:

mysql> use nova;

mysql> select * from instances;

mysql> delete from instances where id = '29';
#ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`nova`.`virtual_interfaces`, CONSTRAINT `virtual_interfaces_ibfk_1` FOREIGN KEY (`instance_id`) REFERENCES `instances` (`id`))

#MySQL 删除 id 为 29 的 instance 时触发外键限制错误，简单的办法是暂时关闭外键检查，等删除后再打开：

mysql> SET FOREIGN_KEY_CHECKS=0;
#Query OK, 0 rows affected (0.00 sec)

mysql> delete from instances where id = '29';
#Query OK, 1 row affected (0.04 sec)

mysql> delete from instances where id = '30';
#Query OK, 1 row affected (0.04 sec)

mysql> SET FOREIGN_KEY_CHECKS=1;
#Query OK, 0 rows affected (0.00 sec)

# 删除 instance 29 和 30后再用 euca-describe-instances 命令验证一下：

euca-describe-instances
```

### 删除`host`为 `NULL` 的实例

```
select uuid,id,host,hostname from instances;
```

发现有很多实例的host是NULL，这些是之前测试的时候创建的，后来被删除了，但是数据库还保留着信息，我想把它删掉：

![image](https://wx2.sinaimg.cn/large/6ee6fa3egy1ftyvgjhg4vj20no08mmxt.jpg)
```
select uuid,id,host,hostname from instances where host='';
select uuid,id,host,hostname from instances where host="NULL";
```
![image](https://ws1.sinaimg.cn/large/6ee6fa3egy1ftyvh14j11j20if01hmx0.jpg)

![image](https://ws1.sinaimg.cn/large/6ee6fa3egy1ftyvhkeez5j20hh019744.jpg)

咦，什么都没删掉？噢 ，想起来了，`NULL`是bool值，只能用 `is` 或 `is not`来匹配：
```bash
MariaDB [nova]> delete from instances where host is NULL;
#ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`nova`.`block_device_mapping`, CONSTRAINT `block_device_mapping_instance_uuid_fkey` FOREIGN KEY (`instance_uuid`) REFERENCES `instances` (`uuid`))
MariaDB [nova]> SET FOREIGN_KEY_CHECKS=0;
#Query OK, 0 rows affected (0.00 sec)

MariaDB [nova]> delete from instances where host is NULL;
#Query OK, 83 rows affected (0.02 sec)

MariaDB [nova]> SET FOREIGN_KEY_CHECKS=1;
#Query OK, 0 rows affected (0.00 sec)

MariaDB [nova]> select uuid,id,host,hostname from instances;
```
![image](https://ws3.sinaimg.cn/large/6ee6fa3egy1ftyvw31xpaj20i8065aaf.jpg)

正常了。



