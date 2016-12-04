---
title: "NFS集成LDAP实现"
date: 2016-11-1 13:31:21
categories: [Linux]
tags: [Linux, ldap, nfs]
toc: true
---

          +---------+    
          |         |        
          |         |        
          +---------+ LDAP服务器
               ^
               |            
               |            
          +---------+        
          |         |        
          |         |
          +---------+ NAS机头
               ^   (LDAP客户端+NFS服务端)
               |
               |
          +---------+        
          |         |        
          |         |
          +---------+ NFS客户端

NFS服务不提供用户级别的权限控制，但能够提供主机级别的权限控制。也就是，允许或禁止给定的主机访问NFS导出的共享目录。LDAP能够提供集中式的主机管理，将一个或多个主机归为一组，称为网络组(netgroup)。NFS集成LDAP的目的，就是利用LDAP的网络组来按组限制主机对共享目录的访问。如上图所示，当NFS服务端接收到NFS客户端挂载共享目录的请求时，先向LDAP服务器询问目标客户机是否在允许的网络组内，如果目标客户机在允许的网络组内则允许挂载，否则禁止挂载。

<!--more-->

## 一个例子

| 主机名 | IP地址 | 角色 | 说明 |
|:--|:--|:--|:--|
| BoreNode1 | 172.16.65.130 | LDAP服务器 | 提供sysadmin网络组|
| BoreNode3 | 172.16.65.142 | LDAP客户端以及NFS服务器 | 提供NFS服务，并根据网络组检验客户机的合法性 |
| BoreNode4 | 172.16.65.143 | NFS客户端 | 属于sysadmin网络组 | 
| BoreNode5 | 172.16.65.144 | NFS客户端 | **不**属于sysadmin网络组 | 

假设NFS允许访问的网络组的名字为sysadmin，上表给出了例子中将用到的4台主机的不同角色，下面逐个介绍不同角色的配置情况。

### LDAP服务器

#### 安装软件

``` shell
root@BoreNode1:~# apt-get install slapd ldap-utils
```
安装结束通过*ps -ef | grep slap*命令检查slapd进程。

#### 重新配置

``` shell
root@BoreNode1:~# dpkg-reconfigure slapd
```
配置过程中的选择：

Omit OpenLDAP server configuration? No
DNS domain name: h3c.com
Organization name: onestor
Database backend to use: BDB
Do you want the database to be removed when slapd is purged? Yes
Move old database? Yes
Allow LDAPv2 protocol? No

#### 基本操作

``` shell
ldapsearch -x -b dc=h3c,dc=com -H ldap://127.0.0.1
```
查看目录树结构

``` shell
ldapdelete -x -D cn=admin,dc=h3c,dc=com -W cn=sysadmin,ou=netgroup,dc=h3c,dc=com
```
删除目录树中的1个节点

``` shell
ldapadd -x -D cn=admin,dc=h3c,dc=com -W -f add_content.ldif
```
从ldif文件添加目录结构


#### 构建数据

```
dc=h3c,dc=com
    |-- cn=admin
    |-- ou=netgroup
        |-- cn=sysadmin
```
目标是构建上面的目录结构，网络组sysadmin节点挂在netgroup部门下面。实际上，按照上述步骤配置LDAP服务器时已经建立了dc=h3c,dc=com和cn=admin,dc=h3c,dc=com两个节点。因此接下来只需要添加ou=netgroup,dc=h3c,dc=com和cn=sysadmin,ou=netgroup,dc=h3c,dc=com两个节点。

```
dn: ou=netgroup,dc=h3c,dc=com                                                                          
objectClass: organizationalUnit                                                                        
ou: netgroup                                                                                           

dn: cn=sysadmin,ou=netgroup,dc=h3c,dc=com                                                              
objectClass: nisNetgroup                                                                               
objectClass: top                                                                                       
cn: sysadmin                                                                                           
nisNetgroupTriple: (BoreNode1,-,-)                                                                     
nisNetgroupTriple: (BoreNode4,-,-)
```
准备内容如上的LDIF文件，将其添加到目录结构。简单介绍下和网络组相关几个schema：
* nisNetgroup 代表一个网络组，必选cn属性，代表网络组的名称。网络组通常作为ou的子节点，本例中挂在ou=netgroup节点下
* nisNetgroupTriple 为网络组的属性，它的值是个三元组(host, user, NIS-domain)，代表1台主机
* memberNisNetgroup 为网络组的属性，代表网络组的子网络组

简而言之，一个网络组可以由一组主机构成，也可以由若干网络组组合而成，还可以由网络组加主机构成。nisNetgroupTriple描述主机，memberNisNetgroup描述子网络组。此外，关于这些预定义schema的更详细内容可以在/etc/ldap/schema目录中找到。

### LDAP客户端

#### 安装软件

``` shell
root@BoreNode3:~# apt-get install ldap-utils libpam-ldap libnss-ldap nslcd
```
安装过程提示的配置：

LDAP server Uniform Resource Identifier: ldap://172.16.65.130/
Distinguished name of the search base: dc=h3c,dc=com
LDAP version to use: 3
Make local root Database admin: Yes
Does the LDAP database require login? No
LDAP account for root: cn=admin,dc=h3c,dc=com
LDAP server URI: ldap://172.16.65.130/
LDAP server search base: dc=h3c,dc=com


#### 认证方式

**修改nsswitch**

``` shell
root@BoreNode5:~# auth-client-config -t nss -p lac_ldap
```
执行上述命令的主要作用是修改/etc/nsswitch.conf文件，下表列出了命令执行前后的该文件的差异。

| 认证项 | 命令前 | 命令后 |
|:--|:--|:--|
| passwd | compat | files ldap |
| group | compat | files ldap |
| shadow | compat | files ldap |
| netgroup | nis  | ldap |

**注意** 上表最后1行并非由auth-client-config命令自动修改，而是手动修改的。

nsswitch是名字服务的开关，决定了查询名字的顺序。以查询passwd为例，修改后将先在本地查找，然后再到ldap中查找。如果删除“files ldap”中的files，那么查询用户时将只从ldap中查找。

``` shell
root@BoreNode3:~# getent netgroup sysadmin
sysadmin              (BoreNode1,-,-) (BoreNode4,-,-)
```
**getent**命令提供了查询各种名字的功能，例如getent passwd用于查询用户，getent netgroup用于查询网络组。但查询网路组时必须提供网络组的名字，因为getent不具备枚举网络组的功能。


**修改slap**

```
nss_base_netgroup   ou=netgroup,dc=h3c,dc=com?one
```
修改/etc/sldap.conf文件，添加上面的配置。


### NFS服务端

**修改hosts文件**

``` shell
root@BoreNode3:~# cat /etc/hosts
127.0.0.1	localhost
172.16.65.130    BoreNode1
172.16.65.142    BoreNode3
172.16.65.143    BoreNode4
172.16.65.144    BoreNode5
```

**修改/etc/exports文件**

```
/root/nfs_1     @sysadmin(rw,sync,no_subtree_check)
```


### NFS客户端

### 主机名问题

问题：如果NFS服务端的/etc/hosts文件不增加网络组的主机的话，那么即使目标客户机属于给定的网络组也无法访问NFS的共享目录。

```
dc=h3c,dc=com
    |-- ou=Hosts
    |   |-- cn=BoreNode1
    |   |-- cn=BoreNode4
    |-- ou=netgroup
        |-- cn=sysadmin
```

解决方法依然是LDAP，如下所示，在目录结构中添加Hosts子树，Hosts子树的每个子节点代表一台主机。配置过程如下：

**LDAP服务器**

Step1: 准备ldif文件，文件内容：

```
dn: ou=Hosts,dc=h3c,dc=com                                                                             
objectClass: organizationalUnit                                                                        
objectClass: top                                                                                       
ou: Hosts                                                                                              
                                                                                                       
dn: cn=BoreNode1,ou=Hosts,dc=h3c,dc=com                                                                
objectClass: ipHost                                                                                    
objectClass: device                                                                                    
objectClass: top                                                                                       
cn: BoreNode1                                                                                          
ipHostNumber: 172.16.65.130                                                                            
                                                                                                       
dn: cn=BoreNode4,ou=Hosts,dc=h3c,dc=com                                                                
objectClass: ipHost                                                                                    
objectClass: device                                                                                    
objectClass: top                                                                                       
cn: BoreNode4                                                                                          
ipHostNumber: 172.16.65.143
```

Step2: 导入ldif文件

**LDAP客户端**

Step1 修改/etc/nsswitch.conf文件

```
hosts:          files ldap
```
查找主机名时，先查找本地文件，然后再到LDAP服务器中查找。

2. 修改/etc/ldap.conf文件

```
nss_base_hosts      ou=Hosts,dc=h3c,dc=com?one
```

3. 验证结果

``` shell
root@BoreNode10:~# getent hosts
127.0.0.1       localhost
172.16.73.233   ubuntu
127.0.0.1       localhost ip6-localhost ip6-loopback
172.16.65.130   BoreNode1
172.16.65.143   BoreNode4
```
输出结果的最后两条记录来自LDAP服务器。

## 参考资料

1. [6.8 Netgroup](http://etutorials.org/Server+Administration/ldap+system+administration/Part+II+Application+Integration/Chapter+6.+Replacing+NIS/6.8+Netgroups/)
2. [LDAP Hosts](https://wiki.archlinux.org/index.php/LDAP_Hosts)

