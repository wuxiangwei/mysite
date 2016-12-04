title: 搭建ocfs2环境
date: 2014-09-19 17:05:11
categories: [Linux]
tags: [OCFS2]
---

OCFS2是基于共享磁盘的集群文件系统，它在一块共享磁盘上创建OCFS2文件系统，让集群中的其它节点可以对磁盘进行读写操作。OCFS2由两部分内容构成，一部分实现文件系统功能，位于VFS之下和Ext4同级别；另一部分实现集群节点的管理。

![](http://images.cnitblog.com/blog/571795/201409/191946064093699.jpg)

如上图所示，测试环境中OCFS2集群由三台服务器组成。我们将cvm服务器中的/dev/sda8分区作为共享磁盘，共享磁盘通过iSCSI共享给client01和client02服务器。

<!--more-->

### 共享磁盘

如果cvm节点中没有单独的磁盘分区，可以参考[Linux命令总结：fdisk](http://www.cnblogs.com/shanno/p/3973366.html)从现有的磁盘分区中划分出一个新分区。测试环境中，我们将cvm节点的/dev/sd8分区作为共享磁盘。磁盘分区通过iSCSI挂给集群中的其它节点，关于iSCSI的用法可以参考[iSCSI:环境搭建](http://www.cnblogs.com/shanno/p/3979675.html)一文。

另外，值得注意的是**cvm节点自己也要通过iSCSI连接共享磁盘，并且mount时挂载iscsi共享磁盘sdb而不是sda8**。否则，在cvm节点写入的数据不能同步到其它节点，并可能导致节点重启。

### 配置集群

OCFS2集群中每个节点的配置都相同，因此只要在其中一个节点中准备好配置，然后scp到其余节点即可。
```
root@cvm:~# cat /etc/ocfs2/cluster.conf 
cluster:
        node_count = 3            <== 集群节点数目
        name = ocfs2              <== 集群名字
node:
        ip_port = 777
        ip_address = 192.168.7.10
        number = 0                <== 节点编号
        name = client01           <== 节点名字
        cluster = ocfs2
node:
        ip_port = 777
        ip_address = 192.168.7.11
        number = 1
        name = client02
        cluster = ocfs2
node:
        ip_port = 777
        ip_address = 192.168.7.8
        number = 2
        name = cvm
        cluster = ocfs2
```
注意：如果粗心大意地将任意两个节点的编号写成一样，那么执行*/etc/init.d/o2cb online ocfs2*命令时会出现**o2cb_ctl: Internal logic failure while adding node cvm**的错误信息。

### 启动OCFS2服务

加载OCFS2服务
```
root@cvm:~# /etc/init.d/o2cb load
Loading filesystem "configfs": OK
Mounting configfs filesystem at /sys/kernel/config: OK
Loading stack plugin "o2cb": OK
Loading filesystem "ocfs2_dlmfs": OK
Creating directory '/dlm': OK
Mounting ocfs2_dlmfs filesystem at /dlm: OK
```

启动集群，只有启动集群后才可以格式化共享磁盘。
```
root@cvm:~# /etc/init.d/o2cb online 
Setting cluster stack "o2cb": OK
Starting O2CB cluster ocfs2: OK
root@cvm:~# /etc/init.d/o2cb start 
```

查看集群状态
```
root@client02:~# /etc/init.d/o2cb status 
Driver for "configfs": Loaded
Filesystem "configfs": Mounted
Stack glue driver: Loaded
Stack plugin "o2cb": Loaded
Driver for "ocfs2_dlmfs": Loaded
Filesystem "ocfs2_dlmfs": Mounted
Checking O2CB cluster ocfs2: Online
Heartbeat dead threshold = 31
  Network idle timeout: 30000
  Network keepalive delay: 2000
  Network reconnect delay: 2000
Checking O2CB heartbeat: Active
```
如果状态中提示**Checking O2CB heartbeat: Not active**信息，那么说明还没挂载共享磁盘。

### 挂载磁盘

在其中一个节点上，将共享磁盘格式化成ocfs2格式。 格式化命令中，-N代表集群允许的最大节点数目。
```
root@cvm:~# mkfs -t ocfs2 -N 4 /dev/sda8
```

在每个节点中将共享磁盘挂载到一个挂载点上，这跟挂载Ext4文件系统一致。测试中，我们将磁盘都挂载到test_ocfs2目录。
```
root@cvm:~# mount -t ocfs2 /dev/sdb test_ocfs2/
root@client01:~# mount -t ocfs2 /dev/sdb test_ocfs2/
root@client02:~# mount -t ocfs2 /dev/sdb test_ocfs2/
```

查看挂载情况
```
root@client01:~# mounted.ocfs2 -f
Device                FS     Nodes
/dev/sdb              ocfs2  client01, client02, cvm
```

最后，在每个节点中创建一个和自己主机名相同的目录，任意一个节点都可以看到其它节点创建的目录。
```
root@cvm:# ls test_ocfs2 
client01  client02  cvm
```
目录client01在主机client01中创建，目录client02在主机client02中创建，目录cvm在主机cvm中创建。但在每个节点中都能够看到其它节点创建的目录。

### 其它

日志路径： /var/log/syslog

![](http://images.cnitblog.com/blog/571795/201409/191946290501081.jpg)

实际应用中，单独使用一台服务器的磁盘总是捉襟见肘，并且存在单点故障的问题。这时，可以通过Ceph提供虚拟共享磁盘(即**rbd块**)给ocfs2集群使用。

参考文献

1. [OCFS2在Linux下的配置文档](http://www.cnblogs.com/zhangpengme/archive/2011/12/29/2306362.html)          
