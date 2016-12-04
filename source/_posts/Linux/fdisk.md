title: "fdisk磁盘分区"
date: 2014-09-15 18:26:31
categories: [Linux]
tags: 
---

fdisk命令能够用来对磁盘进行分区，查看磁盘的分区信息。

<!--more-->

#### 查看分区信息

分区信息包括容量、扇区数目、柱面数目、磁头数目和IO大小等信息。
```
root@cvm:/# fdisk -l /dev/sda7
Disk /dev/sda7: 441.8 GB, 441752485888 bytes
255 heads, 63 sectors/track, 53706 cylinders, total 862797824 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```

磁盘的CHS编址。C代表柱面数目，也就是磁道数目。H代表磁头数目，也就是盘片数目；S代表每磁道的扇区数目。 

总扇区数目 = 磁头数目 x 每个磁道的扇区数目 x 磁道数目:             
255 * 63 * 53706 => 862786890。

磁盘容量的计算：扇区大小 x 扇区数目:             
512 * 862797824 => 441752485888 字节

#### 划分分区

*/dev/sd*代表SCSI设备，*/dev/hd*代表IDE设备。 sda1代表sda盘的一个分区。
```
root@cvm:/# fdisk /dev/sda
```
使用fdisk命令对sda重新分区。

```
Command (m for help): d 
Partition number (1-7): 7
```
删除sda盘的sda7分区。

```
Command (m for help): n
Partition type:
   p   primary (1 primary, 1 extended, 2 free)
   l   logical (numbered from 5)
Select (default p): l
Adding logical partition 7
First sector (113973248-976771071, default 113973248): 
Using default value 113973248
Last sector, +sectors or +size{K,M,G} (113973248-976771071, default 976771071): +150G  
```
创建新分区，分区编号为sda7，分区大小为150G。

```
Command (m for help): w
The partition table has been altered!
```
退出fdisk命令之前，通过*w*命令将分区写入磁盘。

```
root@cvm:/# fdisk -l
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    58593279    29295616   83  Linux
/dev/sda2        58595326   976771071   459087873    5  Extended
/dev/sda5        58595328    97654783    19529728   83  Linux
/dev/sda6        97656832   113971199     8157184   82  Linux swap / Solaris
/dev/sda7       113973248   428546047   157286400   83  Linux
```
查看新分区sda7，它的大小为 *(428546047 - 113973248) \* 512 / (1024 \* 1024 \* 1024) => 150G*。



