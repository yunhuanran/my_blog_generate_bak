title: "OpenStack Cinder 源码解读（基于Mitaka）(一)(源码体系结构)"
date: 2016-12-07 09:31:17
tags: [Openstack,cinder]
---

目前关于cinder的源码解读的中文资料大部分都属于早期OpenStack版本，本文将对较新的Mitaka版本的Cinder除了客户端以外的源码进行解析，主要包括以下内容：

- 代码的体系结构
- 具体源码的注释解读
- 笔记

<!--more-->

## cinder 源码目录结构

### cinder/api

cinder的核心代码之一，对应cinder-api模块，是进入cinder的HTTP接口。

### cinder/backup

cinder的核心代码之一，对应cinder-backup模块，用于提供存储卷的备份功能，支持将块存储被分到Openstak对象存储，比如Swift，Ceph等。

- cinder/backup/drivers
针对不同的存储后端的驱动

### cinder/brick

用于管理本地卷的挂载，目前该部分已经逐渐从cinder中迁移出去，成为独立的项目，项目中遗留的是尚未迁移的部分。
https://github.com/openstack/os-brick

### cinder/cmd

cinder管理的命令行接口，比如cinder各个模块的启动。

### cinder/common

公共代码，包括配置参数信息，数据库工具等。

### cinder/compute

导入Compute API，默认为cinder.compute.nova.API,定义了一些通过Nova客户端实现快照处理等操作的方法。

### cinder/config

该文件夹存放模块的配置文件。

### cinder/consistencygroup

处理一致性组相关的所有请求。（一致性组属于相对较新的功能）

### cinder/db

cinder的数据库的接口，相关数据表的定义与描述。 

### cinder/hacking

用于cinder的特定测试。

### cinder/image

实现使用Glance作为后端的镜像服务，有些操作通过Glance客户端调用Glance中的相应方法实现。 

### cinder/keymgr

秘钥管理。

### cinder/locale

地区化（语言设置）相关的内容。

### cinder/objects

属于较新的组件，将对象引入cinder，组件内部工具api，将各种资源抽象成为对象，对象可以将数据与操作他们的方法捆绑在一起。
通过使用对象，代码将于实际的数据库有一定的隔离，可以使滚动升级更加容易。对象可以用于RPC传递数据，并允许直接从数据库通过RPC延迟加载数据。

### cinder/replication

管理卷的副本。卷的副本可用于HA与容灾恢复。

### cinder/scheduler

cinder的核心代码之一，会根据预定的策略（不同的调度算法），选择合适的cinder-volume节点来处理用户的请求。在用户请求没有指定具体的存储节点时，cinder-scheduler会选择一个合适的节点。如果用户的请求已经指定了具体的存储节点，则直接cinder-scheduler不参与调度，直接交由cinder-volume处理。

### cinder/testing

在项目安装完成之后，里面只有一个关于的简单说明，如果是未安装状态的代码则里面包括了cinder的各种单元测试。

### cinder/transfer

处理卷所有权转换相关的请求，比如卷从一个租户转换到另外一个租户。

### cinder/volume

cinder核心代码之一，运行在存储节点上管理具体存储设备的存储空间，每个存储节点上都会运行一个cinder-volume服务，多个这样的节点一起便构成了一个存储资源池。

- /cinder/volume/drivers/
针对不同存储后端的驱动

### cinder/wsgi

wsgi服务相关的代码。

### cinder/zonemanager

扩展了Cinder对FC的支持。

### cinder

项目根目录下主要包括一些工具类，包括通用工具，版本相关，锁，rpc，ssh工具……等内容。