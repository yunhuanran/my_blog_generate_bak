title: "Ceph更换网络状况处理方法"
date: 2017-5-1 17:50:00
tags: Ceph

---

## 需求
有的时候会遇到部署好的ceph需要更换ip地址的情况，更换过后发现ceph无法正常启动，各种指令无法执行，出现等待超时的错误，本文将给出解决此类问题的教程。
<!--more-->
错误范例：
```
2017-05-08 21:49:23.487685 7f449e7fc700  0 -- :/1009633 >> 192.168.1.186:6789/0 pipe(0x7f4494000c00 sd=3 :0 s=1 pgs=0 cs=0 l=1 c=0x7f4494000e90).fault
2017-05-08 21:49:24.487685 7f449e7fc700  0 -- :/1009633 >> 192.168.1.186:6789/0 pipe(0x7f4494000c00 sd=3 :0 s=1 pgs=0 cs=0 l=1 c=0x7f4494000e90).fault
```

## 操作方法
1.在更换ip前导出mon map
命令如下：
```shell
ceph mon getmap -o map   
monmaptool --print map
```

2.删除旧的配置，将新的ip增加到map,注意下面示例中的node指的是节点的名字
```shell
#方法1：通过monmaptool编辑已有的monmap文件
monmaptool --rm node1 --rm node2 --rm node3 map
monmaptool --add node1 10.0.2.21:6789 --add node2 10.0.2.22:6789 --add node3 10.0.2.23:6789 map

#方法2：通过ceph.conf生成新的monmap文件（不建议，容易出现细节错漏）
sudo monmaptool --create --generate -c /etc/ceph/ceph.conf /etc/ceph/monmap

#方法3：直接创建新的monmap文件（不建议，容易出现细节错漏）
monmaptool --create --add mon.a 101.71.4.20:6789 --add mon.b 101.71.4.21:6789 \
  --add mon.c 101.71.4.22:6789   --add mon.d 101.71.4.23:6789    --add mon.e 101.71.4.24:6789   --fsid c6e7e7d9-2b91-4550-80b0-6fa46d0644f6 \
  --clobber monmap

#查看monmap内容的命令
monmaptool --print map
```

3.复制新的monmap到所有的mon的节点
4.在所有的mon节点更改/etc/ceph/ceph.conf中对应节点的ip地址，可以手动修改，但是更建议通过ceph-deploy推送的所有的ceph节点。
```shell
ceph-deploy --overwrite-conf admin cephmon
```
5.停止所有的mon的进程。
```shell
/etc/init.d/ceph stop mon
```

6.在所有的mon节点上将新的monmap注入配置之中。
```shell
ceph-mon -i node1 --inject-monmap map
```

7.启动monitor
```shell
sudo /etc/init.d/ceph -a start mon.ceph0
```

8.重启所有的osd
```shell
/etc/init.d/ceph restart osd
```

其中7,8两个步骤可以考虑直接重启所有的ceph服务
```shell
/etc/init.d/ceph restart
```