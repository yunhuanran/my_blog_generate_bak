title: "Ceph简单部署易遗漏配置及常见错误"
date: 2015-11-3 17:50:00
tags: Ceph
---
##易忽的略配置

1. 不可以使用root权限进行配置。
2. 如果使用ceph-deploy进行部署需要给部署所用账户root权限，具体方法如下：
>echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
>sudo chmod 0440 /etc/sudoers.d/{username}
<!--more-->

3. 执行命令visudo修改suoders文件：
把Defaults    requiretty 这一行修改为
> Defaults：ceph   ！requiretty

4. hostname要与在集群中的名字一致，否则ceph-deploy会失败。
5. SSH权限不能遗漏。
6. 关闭防火墙。

##常见错误解决方式
###Delta RPMs disabled
 
错误内容：
>Delta RPMs disabled because /usr/bin/applydeltarpm not installed_LinuxEye

解决方法：
>yum provides '*/applydeltarpm'
>yum install deltarpm

###NoSectionError

错误内容：
>[ceph_deploy][ERROR ] RuntimeError: NoSectionError: No section: ‘ceph’

解决方法：
>1. 修改更新源为 eu.ceph.com
>2. 尝试yum remove ceph-release 然后再次执行ceph-deploy

###non-zero exit
错误内容：
>[node1][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: ceph --version

解决方法：
>[root@admin-node ~]# yum install *argparse* -y

如果无效则：
>在每个节点上先行安装ceph

>执行sudo yum -y install ceph

###出错后再次安装失败
ceph-deploy自带的清理功能对于OSD相关的清理效果较好，但是对于mon的清理会不彻底，因此如果mon安装失败要根据错误信息手动清理相关的进程以及文件夹。

##参考资料
>http://www.linuxeye.com/Linux/2726.html
https://www.google.com.hk/
