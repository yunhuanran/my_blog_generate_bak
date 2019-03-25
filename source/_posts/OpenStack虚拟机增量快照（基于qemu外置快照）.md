title: "OpenStack虚拟机增量快照（基于qemu外置快照）"
date: 2015-11-25 19:19:33
tags: [Openstack,qemu,kvm,qemu-kvm,灾备]
---
 
##实践内容
寻找对通过OpenStack建立的虚拟机进行增量快照，并且使得这些快照可以备份在远程的方法。

##技术方案
通过研究了解到Nova自带的快照功能并不支持增量快照，每次快照会将修改的内容与基础镜像合并然后上传到存储后端，然后决定选择特定的hypervisor进行该功能的实现，本文选取qemu作为后端hypervisor。

通过研究发现Centos 7默认的qemu-kvm版本（1.5.3）不支持外部快照，而内部快照的增量对于备份快照并无价值。经过查阅资料决定通过编译源码的方式安装较新的qemu(2.4.1)来进行外部快照。通过外部快照对虚拟机进行快照并得到保存快照信息的镜像文件，最后通过快照合并功能得到需要包含还原信息的镜像从而达到备份的效果。
<!--more-->

##实践步骤
###实验环境搭建
1.通过源码安装新版本的qemu及qemu-kvm
下载qemu源码，qemu-kvm源码并解压：

    git clone git://git.kernel.org/pub/scm/virt/kvm/qemu-kvm.git

qemu官网由于大家都懂的原因无法访问，故手动下载qemu-2.4.1.tar.bz2

    tar jvxf qemu-2.4.1.tar.bz2


2.安装qemu-kvm
```
yum install -y glib*
./configure
make -j 20
make install
```

该步如果出现错误参考qemu安装包解决错误的方法(一般是因为缺少glib和zlib库)。


3.安装qemu依赖包

    yum install gcc libsdl1.2-dev zlib1g-dev libasound2-dev pkg-config libgnutls-dev            pciutils-dev

4.解决编译前置配置的错误：

安装zlib，glib以及libtool

```
yum install zlib*
yum install -y glib*
yum install libtool
```

卸载本机自带的qemu模块

    yum remove qemu
    yum remove qemu-kvm

修改Makefile文件使其在Centos下忽略警告，而不是默认的将警告视为错误

    vi MakeFile
    QEMU_CFLAGS+=-w

（注意添加在相关代码块附近，不可直接添加在开始）

5.编译前的环境配置

    ./configure

6.编译及安装：

    make
    make install

###快照实验
1.下载虚拟机镜像，并存在目录中，并做好创建虚拟机所需的准备。
下载Cirros的镜像作为测试用镜像

    yum install libvirt
    yum install virt-manager virt-viewer
    yum install virt-install

关闭selinux

    # vi /etc/sysconfig/selinux
    SELINUX=disabled
    reboot

注意目录的权限，如/root下的目录会由于目录权限的问题导致虚拟机创建失败。

    mkdir my-cluster
    cd my-cluster
    
2.安装虚拟机:

    virt-install --name=cirros --ram=128 --vcpus=1 --disk /home/images/cirros-0.3.0-x86_64-disk.img,format=qcow2 --import –nonetworks

3.对虚拟机进行快照

    virsh # snapshot-create-as cirros extsnap1 --disk-only --atomic

4.操作虚拟机进行简单的修改，比如添加文件等

5.创建第二个快照

    virsh # snapshot-create-as cirros extsnap2 --disk-only –atomic

6.测试将基础镜像内容合并到快照

```
virsh blockpull --domain cirros --path /home/images/cirros-0.3.0-x86_64-disk.extsnap2
cp cirros-0.3.0-x86_64-disk.extsnap2 testpull.img
virt-install --name=exts2img --ram=128 --vcpus=1 --disk /home/images/testpull.img,format=qcow2 --import –nonetworks
```

经过测试发现可以利用合并后的快照2的镜像生成新的虚拟机，并且虚拟机里保存有修改内容，说明将基础镜像里的内容成功的整合到了快照镜像中。

7.测试将快照内容合并到快照

    cp cirros-0.3.0-x86_64-disk.img cirros01-test.img
    virt-install --name=cosncirros0 --ram=128 --vcpus=1 --disk /home/images/cirros01-test.img,format=qcow2 --import --nonetworks

可以发现利用基础镜像的副本创建的新虚拟机中包含有修改过后的内容，说明快照内容成功的整合到了虚拟机中。

###验证顶层快照无法合并进基础快照

1.选择一个新的镜像创建一个新的虚拟机
2.进行快照

    snapshot-create-as exts2img exts2imgsnap0 --disk-only --atomic

3.修改虚拟机的内容
4.合并快照到基础镜像

    qemu-img commit -f qcow2 testpull.exts2imgsnap0

5.利用基础镜像重新创建一个虚拟机

    virt-install --name=commimg --ram=128 --vcpus=1 --disk /home/images/testcomm.img,format=qcow2 --import –nonetworks

发现新建的虚拟机中没有修改过的内容，说明顶层快照无法合并到基础快照中。

###验证结果
1.	Nova自带的快照功能无法实现增量快照的远程备份。
2.	默认的1.5.1版本的qemu无法进行外部快照。
3.	2.4.1版本的qemu支持外部快照。
4.	进行外部快照操作过后基础镜像会被锁定，不会再被修改，所有的修改将在外部快照的镜像上进行。
5.	可以将基础镜像的内容整合到外部快照中得到新的可以启动虚拟机的镜像。
6.	可以将非顶层快照的内容整合到基础镜像中，得到可以启动虚拟机的镜像。
7.	顶层快照修改的内容无法合并到基础镜像中。