---
title: "dmClock源码分析之编译"
date: 2016-11-10 14:01:26
categories: [mClock]
tags: [mClock]
toc: true
---

编译步骤参考README.md文件。

**问题1**
```
CMake Error at /usr/share/cmake-2.8/Modules/FindPackageHandleStandardArgs.cmake:108 (message):
  Could NOT find gtest (missing: GTEST_LIBRARY GTEST_MAIN_LIBRARY
  GTEST_INCLUDE_DIRS)
```

解决方法
``` shell
apt-get install libgtest-dev 
cd /usr/src/gtest/
cmake CMakeLists.txt 
make
cp *.a /usr/lib/
```
参考 [Getting started with Google Test (GTest) on Ubuntu](https://www.eriksmistad.no/getting-started-with-google-test-on-ubuntu/)


**问题2**
```
CMake Error at /usr/share/cmake-2.8/Modules/FindBoost.cmake:1131 (message):
  Unable to find the requested Boost libraries.
```

解决方法
``` shell
apt-get install libboost-dev
```
