title: Hexo笔记
date: 2015-02-27 21:59:09
categories: [Other]
tags: [Hexo]
---

关于使用Hexo搭建博客的教程已经很多并且足够详细，本文侧重搭建过程中碰到问题的解决。我对博客的要求不高，只要满足下面几点就可以了:

* 不含广告
* 界面足够简洁
* 具备标签、评论、分享功能，支持markdown编辑
* 能使用Git对文章进行版本管理，并且可移植性好


<!--more-->


## 修改本地服务的IP地址

输入hexo s命令启动本地服务。然后，浏览器输入localhost:4000，就能够看到预览效果。服务器的ip地址和端口号在\_config.yaml文件内配置。

```
port: 4000
server_ip: 192.168.7.50
```

如果Linux系统没装图形界面，设置server\_ip为主机IP地址，这样就能够通过本地局域网的其它主机来访问了。

## Git Bash显示中文

打开git bash，控制台输入下面的命令，为ls命令取别名。

```
$ alias ls='ls --show-control-chars --color=auto'
```

## 添加多说评论


Hexo添加多说评论时需要提供[多说](http://www.duoshuo.com)账号的shortname，找了半天没找到这东西。多说没有提供注册账号的入口而是让用户直接以微博、百度等账号登入，登入后在“后台管理”中才有。

## Markdown注意事项

1. 引用内部不要有空行，否则显示有问题。
2. 超链接时，网址一定要以Http打头，不能以www开始。

## 部署到gitcafe

### 生成ssh公钥
```
ssh-keygen -t rsa -C shanno@yeah.net -f ~/.ssh/gitcafe
```
执行上面的命令在~/.ssh目录下生成gitcafe、gitcafe.pub两个文件，它们分别代表私钥和公钥文件。

### 创建gitcafe-pages分支

```
root@taurus:~/test/tiger/.deploy# git remote add origin 'git@gitcafe.com:shannon/shannon.git
root@taurus:~/test/tiger/.deploy# git remote add origin 'git@gitcafe.com:shannon/shannon.git'
root@taurus:~/test/tiger/.deploy# git push -u origin gitcafe-pages 
```

## 参考资料

1. [hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/)
2. [google字库导致hexo modernist首页加载变慢](http://ibruce.info/2013/12/03/fonts-googleapis-lead-to-slow/)
3. [windows下git bash显示中文](http://blog.csdn.net/self001/article/details/7337182)
4. [如何同时使用多个公钥](https://gitcafe.com/GitCafe/Help/wiki/%E5%A6%82%E4%BD%95%E5%90%8C%E6%97%B6%E4%BD%BF%E7%94%A8%E5%A4%9A%E4%B8%AA%E5%85%AC%E7%A7%98%E9%92%A5)
5. [如何将托管在github上的hexo博客转到gitcafe](http://blog.maxwi.com/2014/03/19/hexo-github-to-gitcafe/)

