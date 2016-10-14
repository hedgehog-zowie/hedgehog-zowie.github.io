---
title: shell-trouble-shooting
date: 2016-10-14 17:37:51
toc: true
tags:
- shell
categories: 
- shell
---

记录写shell时遇到的问题、原因及解决办法

# ssh引起的while循环只执行一次

有脚本如下：
```
#!/bin/bash
while read time
do
  echo $time
  ssh 10.10.10.111 "hostname"
done < test.txt
```
test.txt中的内容是：
```
1
2
3
4
5
```
我期望的结果是到10.10.10.111上执行5次hostname命令，但是，运行该脚本仅可执行一次，如下：
```
1
bis-newdatanode-s2b-80
```

## 原因
ssh会读取所有标准输入，如下脚本可以查看ssh中的输入：
```
#!/bin/bash
while read time
do
  ssh 10.10.10.111 "cat"
done < test.txt
```
结果如下：
```
2
3
4
5
```
1已被while读取，所以cat出来还有2、3、4、5，而后while再次读取的时候标准输入已经没有内容，因此while循环只执行了一次便退出。

## 解决方法
查看man ssh，可以得到如下信息：
```
     -n      Redirects stdin from /dev/null (actually, prevents reading from stdin).  This must be used when ssh is run in the background.  A common
             trick is to use this to run X11 programs on a remote machine.  For example, ssh -n shadows.cs.hut.fi emacs & will start an emacs on shad-
             ows.cs.hut.fi, and the X11 connection will be automatically forwarded over an encrypted channel.  The ssh program will be put in the
             background.  (This does not work if ssh needs to ask for a password or passphrase; see also the -f option.)
```
重定向标准输入为/dev/null，可以防止ssh从标准输入中读取数据。
因此在ssh后加上-n参数就可以了，如下：
```
ssh -n 10.10.10.111 "hostname"
```