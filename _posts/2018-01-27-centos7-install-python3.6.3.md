---
layout: post
title:  "CentOS 7 安装Python3.6.3"
description: "本文详细记录了CentOS 7操作系统安装以及配置Python3.6.3的过程，供参考。"
date:   2018-01-27 21:21:01 +0800
categories: Python install
---

## 下载源码包

我们到Python官网下载3.6的源码包`Python-3.6.3.tar.xz`。

将其放到`/home/software`下。

## 解包

解压该文件，并解包，得到`Python-3.6.3`目录：

```
# xz -d Python-3.6.3.tar.xz
# tar -xvf Python-3.6.3.tar
...
# cd Python-3.6.3
```

## 安装依赖包

因为是从源码编译的方式安装，所以我们需要预先安装好环境：

```
# yum install wget gcc make
```

其中

- `wget`是用来下载源码包的，因本次我们已经下载了，所以装不装都无所谓
- `gcc`和`make`用于编译

*Ps.如果用`wget`下载，命令为`wget https://www.python.org/ftp/python/3.6.3/Python-3.6.3.tar.xz`*

同时，还需要先安装部分模块，如下：

```
# 解决 import bz2 报错
# yum install bzip2-devel

# 解决 import curses 报错
# yum install ncurses-devel

# 解决 import sqlite3 报错
# yum install sqlite-devel

# 解决 _dbm _gdbm 缺失提醒
# yum install gdbm-devel

# 解决 _lzma 缺失提醒
# yum install xz-devel

# 解决 _tkinter 缺失提醒
# yum install tk-devel

# 解决 readline 缺失提醒以及方向键行为非预期的问题
# yum install readline-devel
```

## 编译安装

我们接上面，解包之后，先配置，再编译：

```
# pwd
/home/software/Python-3.6.3
# ./configre --prefix=/usr/local/python3.6 --enable-optimizations
...
```

上面命令中的参数，`--prefix`指定预期安装的目录，`--enable-optimizations`是优化选项（LTO，PGO等），加上这个flag编译后，性能有10%左右的优化，但是会明显的增加编译时间。
具体我们不讨论，参见（https://gcc.gun.org/onlinedocs/gccint/LTO-Overview.html）

然后再：

```
# make
...
# make install
...
```

安装完成。

## 修改链接

安装完成后，我们可以检查：

```
# /usr/local/python3.6/bin/python3 --version
Python 3.6.3
```

即表明安装成功了，但是我们运行`python --version`，仍然显示2.x的版本：

```
# python --version
Python 2.7.5
```

这是因为默认的符号链接`/usr/bin/python`还是指向Python2.7.5的，我们需要将其修改，如下：

```
# ln -s /usr/local/python3.6/bin/python3 /usr/bin/python3
# rm /usr/bin/python
# ln -s /usr/bin/python /usr/bin/python3
```

现在再运行`python --version`就对了。

## 后续问题

我们经过上面安装好Python3.6.3，并配置好后，会发现`yum`命令用不了了，报错如下：

```
#  yum list
  File "/bin/yum", line 30
    except KeyboardInterrupt, e:
                            ^
SyntaxError: invalid syntax
```

这是因为`yum`包使用Python2.x来开发的，我们将`/usr/bin/python`链接到Python3.x之后，就有问题了。

解决方法也很简单，因为我们Python2.x的版本还在，并没有删除，所以修改如下两个文件头中的申明，将`#!/usr/bin/python`改为`#!/usr/bin/python2.7`即可：

```
# vi /usr/bin/yum

# vi /usr/libexec/urlgrabber-ext-down
```

> 遇到类似的问题报错，可以检查一下对应的文件是否因为这个问题引起的。
> 当然，更好的解决方法当然是不修改`/usr/bin/python`链接，这纯粹是作si的搞法...
