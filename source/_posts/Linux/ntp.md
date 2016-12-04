title: "ntp时间同步"
date: 2014-12-01 19:40:45
categories: [Linux]
tags: 
---

同步时间是分布式系统中最基础的要求，不论是openstack还是ceph都必须先让节点间的时间保持一致。

<!--more-->

下载ntp软件包
```
root@rgw01:~# apt-get install ntp

```

调整ntp server时间
```
root@rgw01:~# date
Mon Dec  1 17:02:03 EST 2014
```

修改ntp服务配置
```
root@rgw01:~# cat /etc/ntp.conf 
restrict 192.168.1.0 mask 255.255.0.0 nomodify
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```
重启ntp服务
```
root@rgw01:~# service ntp restart 
```

客户端设置定时器，定时同ntp服务端同步时间。
```
root@snode01:cat /etc/crontab 
# m h dom mon dow user  command
  1 *  *   *   *  root  /usr/sbin/ntpdate 192.168.7.201
```



