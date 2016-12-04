title: Python笔记
date: 2014-07-31 09:13:43
categories: [Python]
tags: [Python]
toc: ture
---

记录使用python过程中碰到的小问题、小窍门。

<!--more-->

### 剔除字符串末尾的空白字符

```python
with open(pkglist, 'r') as fd:
   for line in fd.readlines():
    if not line.isspace():
       cmdstr = "dpkg -i '%s'" % os.path.join(pkgpath, line.rstrip())
       os.system(cmdstr)
```
将命令写在文件中，然后逐行读取文件中的命令执行。由于从文件中读取出来的行都带有换行符号，所以执行命令时会出错，这时候可以使用字符串**str**类的*rstrip()*方法将行末尾的空白字符剔除。


### 中文乱码

```python
import sys  
reload(sys)  
sys.setdefaultencoding('utf-8')
```
在出现乱码的地方添加上面的代码。


### with用法和异常处理

```python
# !/usr/bin/env python  
# -*-coding:utf8-*-  
  
# 为文件中每行的开头添加行号  
def addLineNumberForFile(srcFilePath, dstFilePath):  
    if srcFilePath == dstFilePath:  
        raise NameError('Invalid arguments, ' + 'p1 = ' + srcFilePath + ', p2 = ' + dstFilePath)  
  
    with open(srcFilePath) as srcfd:  
        lineNum = 0  
        with open(dstFilePath, 'w') as dstfd:  
            for line in srcfd.readlines():  
                lineNum += 1  
                dstfd.write(str(lineNum) + "\t" + line)  
  
# 单元测试  
if __name__ == '__main__':    
    srcFilePath = 'data.txt'  
    dstFilePath = 'data_latest.txt'  
    try:  
        addLineNumberForFile(srcFilePath, dstFilePath)  
        print('Success to add line number for the file named ' + srcFilePath)  
        print('The result file is ' + dstFilePath)  
    except NameError as nError:  
        print nError  
```
