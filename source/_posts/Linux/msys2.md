---
title: "msys2安装配置"
date: 2016-10-11 9:47:45
categories: [Linux]
tags: [Linux, msys2]
toc: true
---

需求：使用MinGW工具链编译出原生的Window应用程序。

[MSYS2](https://sourceforge.net/p/msys2/wiki/MSYS2%20introduction/)由3个子系统组成：msys2、mingw32、mingw64。msys2子系统提供软件编译、软件包管理以及shell功能。

<!--more-->

## MSYS2安装

参考
http://msys2.github.io/

安装结束
msys2安装目录和普通的Linux目录相同。

## Pacman软件包管理

**启动**
进入msys2安装目录（D:\msys64），双击msys2.exe文件，进入交互界面。

**修改源**（可选）
参考 https://lug.ustc.edu.cn/wiki/mirrors/help/msys2
不修改源，经测试，速度还行

**配置代理**
```
export http_proxy=http://172.16.31.2:808 && export https_proxy=http://172.16.31.2:808
```

**pacman详细用法**
参考 https://wiki.archlinux.org/index.php/pacman

### 更新软件包

```
pacman -Syu
```

### 列出已安装软件

```
pacman -Qe
```

### 搜索软件包

```
pacman -Ss mingw | grep gcc
```

### 安装软件

```
pacman -S mingw64/mingw-w64-x86_64-gcc
```
安装mingw-gcc软件包

### 卸载软件

```
pacman -R gcc
```
卸载gcc软件包


## 准备开发环境

### 安装mingw-gcc

```
pacman -S mingw-w64-x86_64-gcc
```
注意，msys2存在msys和mingw两种版本的gcc软件包，默认是msys版本的软件包。该软件包编译出的应用程序依赖msys-2.0.dll文件，因此安装mingw64版本的gcc软件包。安装mingw版本的gcc时，先重命名msys2安装目录下的mingw64.exe和mingw64.ini两个文件(本文在这两个文件名前添加_前缀，如果不修改mingw64.ini文件，也可以正常安装gcc软件包，但无法执行重命名后的mingw64.exe文件)，否则将报错。

安装完 mingw-w64-x86_64-gcc后，通过_mingw64.exe文件启动交互界面，否则如果通过msys2.exe启动交互界面，将发现gcc命令无法执行的错误。

**目标**：测试gcc是否正常工作
创建如下工程：
```
$ tree demo
demo
└── demo.c
```

demo.c文件的内容：

```C
#include <stdio>

int main(int argc, char *argv[])
{
    printf("Hello, baby");
    return 0;
}
```

编译工程：

```shell
gcc demo.c -o demo.exe
```

双击demo.exe文件，不会提示依赖 msys-2.0.dll文件，可以正常运行。

### 编译ceph-dokan

**编译boost库**

[下载](https://sourceforge.net/projects/boost/?source=typ_redirect)boost1.60.0代码

```
unzip boost_1_60_0.zip
cd boost_1_60_0/
./bootstrap.bat mingw
./b2 toolset=gcc --with-system
```
此处，只编译boost的system模块


**编译ceph-dokan**

修改Makefile，指定boost路径

修改编译选项 -fpermi

错误1：

```
In file included from ./include/int_types.h:30:0,
                 from ./include/types.h:18,
                 from auth/Crypto.h:18,
                 from libcephfs.cc:20:
./mingw_include/linux/types.h:13:24: fatal error: parts/time.h: No such file or directory
 #include <parts/time.h>
                        ^
compilation terminated.
make: *** [Makefile:13：libcephfs.o] 错误 1
```



