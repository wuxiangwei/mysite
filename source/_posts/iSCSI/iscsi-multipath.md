title: iSCSI 多路径
date: 2014-09-18 08:50:10
categories: [iSCSI]
tags: [iSCSI]
---


![](http://images.cnitblog.com/blog/571795/201410/081927249055512.jpg)

上图(a)给出了计算机的总线结构，SCSI磁盘挂在SCSI卡上，SCSI卡和网卡都挂接在PCI总线。向SCSI磁盘读写数据时，由SCSI驱动程序生成SCSI指令并发送到挂接在PCI总线上的SCSI卡，SCSI卡翻译指令驱动硬盘进行读写操作。如果将由SCSI驱动程序生成的SCSI指令封装成报文通过网络传输到另外一台计算机，由远端计算机从报文中解析出SCSI命令，并将SCSI命令应用在自己的本地硬盘上，这就实现了iSCSI的功能。如图(b)所示，绿色的曲线表示读写数据的计算机(initiator端)将SCSI指令封装成报文通过网卡发送出去，而接受报文的计算机(target端)解析出报文中的SCSI指令后通过红色曲线读写本地的SCSI磁盘。

<!--more-->

![](http://images.cnitblog.com/blog/571795/201410/081927462181456.jpg)

图(c)给出了iSCSI多路径的实现。Initiator端通过多个网卡将数据发送到Target端，充分利用多网卡提高读写远端磁盘的效率。Initiator端通过多个网卡将Target端的Lun挂接到本地，如上图中的sdb和sdc。要实现多个网卡共同分担读写磁盘的负载，还需要将这两个Lun合并成一个Lun，即sde。这需要通过multipath工具来实现。Multipath不仅实现了I/O流量的负载分担，还实现了故障切换和恢复。当一块网卡故障时，它可以只通过另一块网卡传输数据。

**参考资料**

1. [iSCSI多路径实现KVM高可用](http://www.ibm.com/developerworks/cn/linux/1303_zhuzq_iscsikvmha/)    
2. [Multipath实现LUN设备名称的持久化](http://blog.csdn.net/tianlesoftware/article/details/5979061)          
3. [iSCSI Network Designs: Part5-iSCSI Multipathing,Host Bus Adapters,High Availability and Redundancy](http://etherealmind.com/iscsi-network-design-part-5-iscsi-multipathing-hba-ha-high-availability-redundancy/)    
4. [Linux 内核中的 Device Mapper 机制](http://www.ibm.com/developerworks/cn/linux/l-devmapper/index.html)          
