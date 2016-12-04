title: ocfs2使用ceph块设备
date: 2015-03-05 12:50:57
categories: [Linux]
tags: [OCFS2, Ceph, iSCSI]
---


系统最底层是ceph，ceph之上是tgt，tgt为ceph的rbd块提供iSCSI接口。最后，将iSCSI的lun作为ocfs2的共享磁盘。实际上，也可以不使用tgt，直接将rbd块通过内核模块映射到本地后作为ocfs2的共享磁盘。

<!--more-->

### tgt连接ceph块

```
<target iqn.2015-02.onestore.com:tc>
    driver iscsi
    bs-type rbd             # lun类型为rbd
    backing-store test/tc05 # 导出ceph的test pool中tc05、tc06两个块
    backing-store test/tc06
</target>
```
在tgt的配置文件中增加上述配置，然后重启tgt即可。重启tgt后，使用tgt-admin -s命令查看target导出的lun。

### 关闭rbd缓存

关闭rbd缓存，否则mount文件系统时会报下面的错误。因为在当前节点格式化共享磁盘后，ocfs2文件系统的信息没有完全写入到磁盘而是停留在rbd客户端，其它节点无法感知。
```
internal error Child process (/bin/mount -t ocfs2 /dev/disk/by-path/ip-127.0.0.1:3260-iscsi-iqn.2015-02.onestore.com:tc-lun-1 /vms/w01) unexpected exit status 1: mount.ocfs2: Stale NFS file handle while mounting /dev/sdh on /vms/w01. Check 'dmesg' for more information on this error.
```

关闭rbd缓存的方法是，在ceph.conf文件中添加下面配置。默认情况下缓存是开启的。
```
[client]
rbd cache = false
```

### 参考资料

1. [Start With the RBD Support for TGT](http://www.sebastien-han.fr/blog/2014/07/07/start-with-the-rbd-support-for-tgt/) 

