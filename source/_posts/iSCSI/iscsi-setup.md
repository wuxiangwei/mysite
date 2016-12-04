---
title: 搭建iSCSI环境
date: 2014-09-18 17:00:18
categories: [iSCSI]
tags: iSCSI
toc: true
---

组网环境

```
+----------+---------------+---------------+
| hostname |  ip address   |  iscsi role   |
+----------+---------------+---------------+
| cvm      |  192.168.7.8  |    target     |
| client01 |  192.168.7.10 |    initiator  |
| client02 |  192.168.7.11 |    initiator  |
+----------+---------------+---------------+
```
机器cvm将本机上的/dev/sda8磁盘通过iscsi共享出去，机器client01和client02连接到共享的磁盘。

<!--more-->

### iSCSI target侧

修改tgt配置文件/etc/tgt/target.conf，将/dev/sda8共享出去。注意，target名称个格式为**iqn.yyyy-mm.<reversed domain name>:identifier**，即以iqn开头，后接日期和反转域名。identifier为target的标识，可以自己取，这里我们取为test-tgt。
```
root@cvm:~# cat /etc/tgt/targets.conf
<target iqn.2014-09.com.h3c.cvm:test-tgt>
        backing-store /dev/sda8
</target>
```

配置文件修改完成后，重启tgt服务。tgtd默认监听3260端口。
```
root@cvm:~# service tgt restart        <== 重启target服务
root@cvm:~# netstat -npl | egrep 3260
tcp        0      0 0.0.0.0:3260            0.0.0.0:*               LISTEN      686/tgtd        
tcp6       0      0 :::3260                 :::*                    LISTEN      686/tgtd   
```

查看配置文件共享出去的Lun的信息。**注意，共享出去的分区*/dev/sda8*不能被使用，否则看不到相应的LUN**。
```
root@cvm:~# tgt-admin -s
Target 1: iqn.2014-09.com.h3c.cvm:test-tgt
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Readonly: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
                LUN: 1                    <== 如果/dev/sda8被使用，那么这里不会显示LUN:1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 280690 MB, Block size: 512
            Online: Yes
            Removable media: No
            Readonly: No
            Backing store type: rdwr
            Backing store path: /dev/sda8
            Backing store flags: 
    Account information:
    ACL information:
        ALL
root@cvm:~# 
```
查询连接到target端的initiator节点，使用tgtadm --lld iscsi --op show --mode target命令。

### iSCSI initiator侧

发现iscsi target
```
root@client01:~# iscsiadm -m discovery -t st -p 192.168.7.8
192.168.7.8:3260,1 iqn.2014-09.com.h3c.cvm:test-tgt
```

连接设备
```
root@client01:~# iscsiadm -m node iqn.2014-09.com.h3c.cvm:test-tgt -p 192.168.7.8 -l
Logging in to [iface: default, target: iqn.2014-09.com.h3c.cvm:test-tgt, portal: 192.168.7.8,3260] (multiple)
Login to [iface: default, target: iqn.2014-09.com.h3c.cvm:test-tgt, portal: 192.168.7.8,3260] successful.
```

查询挂接到本地的iscsi磁盘，因为本地已经有一块iscsi磁盘sda了，所以新的磁盘名字为sdb。
```
root@client01:~# ls -l /dev/disk/by-path/
lrwxrwxrwx 1 root root 9 Sep 18 04:15 ip-192.168.7.8:3260-iscsi-iqn.2014-09.com.h3c.cvm:test-tgt-lun-1 -> ../../sdb
```

查看iscsi连接状态
```
root@client01:~# iscsiadm -m node -S
192.168.7.8:3260,1 iqn.2014-09.com.h3c.cvm:test-tgt
```

卸载设备
```
root@client01:~# iscsiadm -m node -T iqn.2014-09.com.h3c.cvm:test-tgt -u
Logging out of session [sid: 3, target: iqn.2014-09.com.h3c.cvm:test-tgt, portal: 192.168.7.8,3260]
Logout of [sid: 3, target: iqn.2014-09.com.h3c.cvm:test-tgt, portal: 192.168.7.8,3260] successful.
```

### initiator 权限

修改tgt的配置文件，可以指定能够连接到target的initiator。如下，只允许192.168.7.11的initiator节点连接到target。
```
root@cvm:~# cat /etc/tgt/targets.conf
<target iqn.2014-09.com.h3c.cvm:test-tgt>
        backing-store /dev/sda8
        initiator-address 192.168.7.11      <== 只准许7.11的节点连接到target
</target>
```

分别在client01和client02上连接target，只有client02可以连接到target。
```
root@client01:~# iscsiadm -m discovery -t st -p 192.168.7.8 
iscsiadm: No portals found

root@client02:~# iscsiadm -m discovery -t st -p 192.168.7.8
192.168.7.8:3260,1 iqn.2014-09.com.h3c.cvm:test-tgt
```
如果要使client01和client02都可以连接到target，那么只需要修改配置文件的initiator-address为192.168.0.0/16即可。

### 其它

1、tgt的日志信息位于/var/log/syslog文件             
2、查看initiator的名字。名字记录在*/etc/iscsi/initiatorname.iscsi*文件中          

参考资料

1. [网络驱动器装置： iSCSI 服务器](http://vbird.dic.ksu.edu.tw/linux_server/0460iscsi.php)           
2. [ISCSI学习文档](http://blog.sina.com.cn/s/blog_755da69701014ckv.html)


