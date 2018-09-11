---
title: "在 OpenStack Ocata/Queens 上一键完成实例疏散"
excerpt: 当物理服务器宕机又无法正常开机，我们就需要将原来运行在这台服务器上的虚拟机，快速迁移到其它正常的物理服务器上，迅速恢复服务，实现服务的高可用。
permalink: /openstack/evacuate-instances-on-openstack-ocata-and-queens
last_modified_at: 2018-08-23
layout: collection
#header:
#  image: /assets/images/openstack.jpg
#author_profile: true

tags:
  - openstack
  - ocata
  - queens
  - evacuate
  - migrate
---

### 概念梳理

先明确几个概念。

什么是实例疏散？

如果硬件故障或其他错误导致OpenStack计算节点启动失败,我们就可以使用实例疏散让在其它可用节点重启这些实例，恢复它们的服务。

随着历史的推演，openstack 的文档和命令中有些概念趋于模糊和混淆，我们常常弄不清，比如 instance 和 server，host 和 node，migration 和 evacuate ，它们有什么区别的呢？

ServerFault 上有人提了问，然后有大神做了详细的解释，引用一下，同时无比感激。

> https://serverfault.com/a/761882/368226

![image](https://ws1.sinaimg.cn/large/6ee6fa3egy1fujm5iikssj20my0lzwij.jpg)


做个简要整理：

- 在 OpenStack 中，`server`指的是 `instance` (实例)，host 指的是物理主机，如：  

  `computer node`、`controller node`、`network node`、`storage node` 等；

- 关于实例疏散和迁移，分两种情况：

  - 计算节点宕机无法恢复

    1. 执行 `nova evacuate` 在其它主机启动一个新的实例
    2. 执行 `nova host-evacuate` 在其它主机启动全部实例

  - 计算节点正常运行  
    1. 执行 `nova host-evacuate-live` 尝试热疏散所有实例到其它主机
    2. 执行 `nova host-servers-migrate` 迁移所有停机的实例到其它主机
    3. 执行 `nova live-migration` 热迁移单个实例到其它主机
    4. 执行 `nova migrate` 迁移单个停机的实例到其它主机



### 疏散单个实例

openstack 主机疏散的文档非常简单，它只写了疏散单个实例的功能

> https://docs.openstack.org/nova/queens/admin/evacuate.html

但 `nova` 帮助文档提供了更详细的用法：

![image](https://wx1.sinaimg.cn/mw690/6ee6fa3egy1fujnp7zs00j20ne0cq757.jpg)

示例操作
```
# 列出故障计算节点的所有实例
nova list --host computer01

# 执行疏散实例，让 scheduller 自动调度到其它节点
nova evacuate instance_id

# 也可指定目标主机
nova evacuate instance_id --target_host computer02
```




### 疏散全部实例

实际上 nova 还有疏散整个故障主机上全部实例的功能，文档参考红帽的：

> https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/instances_and_images_guide/ch-manage_instances#section-move-all-instances

`nova` 帮助文档也提供了稍微详细的用法：

![image](https://ws4.sinaimg.cn/mw690/6ee6fa3egy1fujnpkgta3j20pl0enjse.jpg)

示例操作
```
# 让 scheduller 自动调度
nova host-evacuate computer01

# 自己指定目标主机（如新增的集群节点）
nova host-evacuate computer01 --target computer02
```

### 手动疏散（更新数据库）

详细参考

> https://docs.openstack.org/nova/queens/admin/node-down.html
