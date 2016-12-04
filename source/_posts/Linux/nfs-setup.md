title: 搭建nfs环境
date: 2014-09-06 17:15:45
categories: [Linux]
tags: nfs
---


通过VirtualBox创建两台虚拟机client1和client2，这两台虚拟机和物理主机组成一个网络。将物理主机作为NFS的服务端，虚拟机client1和client2作为NFS的客户端。 物理主机装Mint系统，虚拟机中装Fedora 20系统。

<!--more-->


### 修改client的运行级别

安装的Fedora系统启动时默认进入图形界面，由于图形界面比较消耗资源，并且我电脑的性能有限。为此，希望系统启动时可以进入控制台界面。

### 修改虚拟机的网络连接

VirtualBox创建的虚拟机默认使用NAT方式连接网络，这种模式下虚拟机可以访问主机，并且通过主机访问外网。但是，主机不能访问虚拟机，虚拟机之间也不能相互访问。修改虚拟机的连接方式为桥接模式，这种方式虚拟机可以直接连接到主机连接的交换机上。

设置方法参考[《VirtualBox网络设置详解》](http://reverland.bitbucket.org/VirtualBox_net.html)。


## NFS服务端配置

### 安装nfs-kernel-server

```
sudo apt-get install nfs-kernel-server
shanno@taurus-p245 ~ $ nfs
nfsdcltrack  nfsidmap     nfsiostat    nfsstat
```
安装完成后，可以发现多了上面这些和nfs相关的可执行命令。

### 指定共享目录

```
shanno@taurus-p245 ~ $ cat /etc/exports 
# /etc/exports: the access control list for filesystems which may be exported
#       to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/home/shanno/vshared *(insecure,rw,sync,no_root_squash)
```
指定共享目录以及访问权限。


```
shanno@taurus-p245 ~ $ sudo /etc/init.d/nfs-kernel-server restart
shanno@taurus-p245 ~/tmp $ sudo mount -t nfs localhost:/home/shanno/vshared vshared_nfs/
```
重启nfs进程，并在本地挂载测试。


## NFS客户端操作

### 查看NFS共享目录

```
[root@client1 ~]# rpm -q showmount
package showmount is not installed
[root@client1 ~]# yum install showmount
```
注意：fedora查询命令是否已经安装和安装命令的命令。

```
[root@client1 ~]# showmount -e 192.168.1.102
Export list for 192.168.1.102:
/home/shanno/vshared 192.168.1.*
```
通过showmount命令，我们可以查看给定的NFS服务器(192.168.1.102)导出的共享目录。

### 挂载共享目录到本地

```
[root@client2 temp]# pwd
/root/temp
[root@client2 temp]# ls
vshared_nfs
[root@client2 temp]# mount -t nfs 192.168.1.100:/home/shanno/vshared vshared_nfs/
```
将NFS共享目录挂载到本地的*/root/temp/vshared_nfs*目录，使用*mount*命令挂载时通过*-t*选项指定文件系统类型为nfs。

```
[root@client2 temp]# mount -t nfs 192.168.1.100:/home/shanno/vshared/白马湖车展 vshared_nfs/
```
我们也可以将NFS导出目录下的某个子目录挂载到本地，如上，将导出目录下的*白马湖车展*子目录挂载到*/root/temp/vshared_nfs*目录。

```
[root@client2 vshared_nfs]# mount | egrep nfs
192.168.1.100:/home/shanno/vshared on /root/temp/vshared_nfs type nfs4 (rw,relatime,vers=4.0,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.103,local_lock=none,addr=192.168.1.100)
```
查看nfs目录挂载情况。

**参考资料**

1、[鳥哥的 Linux 私房菜](http://linux.vbird.org/linux_server/0330nfs.php)



