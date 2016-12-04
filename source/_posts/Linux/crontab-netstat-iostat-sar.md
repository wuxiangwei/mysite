title: "Linux几个流量监控命令"
date: 2014-08-29 17:35:27
categories: [Linux]
tags:
---

总结几个监控IO流量的命令。

<!--more-->

## crontab
cron可以设定在指定的时间运行任务。

**1、查看定时任务**

```
[root@client1 ~]# crontab -l -u root
*/1 * * * * date >> /root/1.txt
[root@client1 ~]# cat /var/spool/cron/root 
*/1 * * * * date >> /root/1.txt
```

查看root用户的定时任务。上面的例子中只有一个定时任务，它每隔1分钟时间将当时的时间追加到*/root/1.txt*文件。实际上，定时任务被写入到*/var/spool/cron/*目录中。

**2、编辑定时任务**
```
crontab -e 
```
编辑定时任务。

```
[root@client1 ~]# cat /etc/crontab 
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
```
定时任务的设置格式如上所示，它包含执行命令的时间以及命令的内容两部分内容。命令执行时间分成分、时、天、月以及星期5个部分。实际上，cron进程除了从*var/spool/cron*目录中读取定时任务外，还从*/etc/crontab*中读取。后者是针对系统的定时任务，因此命令执行时间和命令内容外还多了项用户。

**3、cron进程**
```
[root@client1 ~]# /etc/init.d/crond restart
Stopping crond: [  OK  ]
Starting crond: [  OK  ]
```
修改定时任务后，可以不重启cron进程。

## iostat
用于打印块设备、分区以及NFS的IO的统计信息。此外，还会打印CPU的统计信息。

**1、用法**
```
[root@client1 ~]# iostat -d /dev/sda1
Device:            tps   Blk_read/s   Blk_wrtn/s   Blk_read   Blk_wrtn
sda1              0.02         0.69         0.00      21232         32
[root@client1 ~]# iostat -d /dev/sda1 -k
Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda1              0.02         0.34         0.00      10616         16
[root@client1 ~]# iostat -d -x /dev/sda1 10 2
Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda1              0.01     0.00    0.02    0.00     0.77     0.00    32.41     0.00    0.68   0.52   0.00
Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
```
例子打印了2次*/dev/sda1*块设备的IO统计信息，每次间隔为*10*秒钟。命令参数*-d*代表值输出设备的IO信息不打印CPU的统计信息；*-x*代表打印扩展信息; *-k*代表以kB_read的格式输出。

**2、输出说明**

*一般信息(读写数据量)*

tps: 每秒发送到设备的transfer数目，transfer由多个逻辑的请求合并而成，它的大小是不确定的。

Blk_read/s:  每秒从设备中读取的数据的大小，以块为单位。块大小和文件系统有关，一般为4K字节。        
Blk_write/s: 每秒写入到设备的数据量，以块为单位。

Blk_read: 从设备读取的Block数目；注意这不是每秒的平均值，而是整个测试过程。
Blk_wrtn: 写入到设备的Block数目；

*扩展信息(IO请求)*

rrqm/s: r(read)rq(request)m(merge)  每秒合并的读请求数目;     
wrqm/s: w(write)rq(request)m(merge) 每秒合并的写请求数目;

r/s:  r(read)  每秒发送到设备的读请求数目;      
w/s:  w(write) 每秒发送到设备的写请求数目；

rsec/s:  r(read)sec(sector)  每秒读扇区的数目；       
wsec/s:  w(write)sec(sector) 每秒写扇区的数目；

avgrq_sz: avg(average)rq(request)-sz(size) 发送到扇区的请求的平均大小；       
avgqu_sz: avg(average)qu(queue)-sz(size)   请求队列的平均大小；

await:  发送到设备的IO的平均时间，包括请求在队列中的等待时间和请求处理时间两部分。时间单位是毫秒(millisecond)      
util:  IO请求发送到设备占用的CPU时间。当这个值接近100%时，说明设备接近饱和。

## netstat

打印网络链接、路由表、接口统计等信息。

**1、连接的协议类型**

-t：TCP连接        
-u: UDP连接

**2、连接状态**

-l: 只打印监听状态的连接；    
-a: 显示所有状态的连接，默认不打印监听状态的连接。

```
shanno@taurus-p245 ~ $ netstat -tnp        ## 默认不打印监听端口
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 192.168.1.104:44531     203.208.41.153:80       ESTABLISHED 13765/firefox   
shanno@taurus-p245 ~ $ netstat -tnpl       ## 只打印监听端口 
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN      -               
    tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -               
shanno@taurus-p245 ~ $ netstat -tnpa       ## 全打印，包括监听端口
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -               
    tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -               
    tcp        0      0 192.168.1.104:44531     203.208.41.153:80       ESTABLISHED 13765/firefox   
```

**3、其它**

-p: 进程
-c: 每隔1秒钟持续输出连接状态
-n: 以点分四段（例如192.168.0.1）的格式打印IP地址

**4、用法**

a. 通过端口号查询进程号；
b. 查询进程的端口号。

## sar

打印系统的活动信息，用于监视网络流量、磁盘IO等。

**1、安装**

sudo apt-get install sysstat 

**2、命令格式**

sar [options] [interval [count]]

interval代表采样间隔，count代表采样次数。

**3、监视网络流量**

```
shanno@taurus-p245 ~ $ sar -n DEV 3
00时08分52秒     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
00时08分55秒      eth0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
00时08分55秒        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
00时08分55秒     wlan0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
00时08分55秒     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
00时08分58秒      eth0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
00时08分58秒        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
00时08分58秒     wlan0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
Average:         eth0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:        wlan0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```
例子中-n代表network，它后面可以跟DEV，代表网络设备。

**错误信息**       
```
root@cvknode20221:~# date
Tue Sep  2 11:38:26 CST 2014
root@cvknode20221:~# sar -n DEV
Invalid system activity file: /var/log/sysstat/sa02
```

**解决方法**     
```
root@cvknode20221:~# sar -o 02
root@cvknode20221:~# ls /var/log/sysstat/
sa02
root@cvknode20221:~# sar -n DEV
11:38:46 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
11:38:48 AM     vnet4      0.00     45.50      0.00      8.82      0.00      0.00      0.00
11:38:48 AM     vnet1      0.00     45.50      0.00      8.82      0.00      0.00      0.00
11:38:48 AM      eth0     52.50      3.00      9.68      0.34      0.00      0.00      8.50
11:38:48 AM      eth1      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```
