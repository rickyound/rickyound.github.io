---
layout: post
author: "杨小定定"
title:  "CentOS安装MySQL以及防火墙配置开放3306端口"
description: "过程记录。"
date:   2018-05-16
categories: technology
---

> 环境
> 1. VMware 14 pro
> 2. CentOS 7
> 3. 虚拟机网络选NAT模式
> 4. MySQL 5.6.40

## 准备

官网下载[mysql-5.6.40-linux-glibc2.12-x86_64.tar.gz](https://dev.mysql.com/downloads/mysql/)。

使用`root`用户登陆CentOS（或普通用户登陆，然后切换`root`也行）。

## 安装

**1. 新增用户和组**

```
# groupadd mysql
# useradd -g mysql mysql
```

**2. 解压`tar`包**

```
# tar -zxvf mysql-5.6.40-linux-glibc2.12-x86_64.tar.gz
```

**3. 复制到安装目录**

```
# cp mysql-5.6.40-linux-glibc2.12-x86_64 /usr/local/mysql
```

**4. 更换工作目录并创建mysql数据目录**

```
# cd /usr/local/mysql
# mkdir ./data/mysql
```

**5. 更改目录属主为`mysql`**

```
# chown -R mysql:mysql ./
```

**6. 执行安装，指定用户和数据目录**

```
# ./scripts/mysql_install_db --user=mysql --datadir=/usr/local/mysql/data/mysql
```

**7. 复制启动脚本并修改权限**

```
# cp support-files/mysql.server /etc/init.d/mysqld
# chmod 755 /etc/init.d/mysqld
```

**8. 复制配置文件**

```
# cp support-files/my-default.cnf /etc/my.cnf
```

**9. 修改启动脚本**

```
# vi /etc/init.d/mysqld

basedir=/usr/local/mysql/
datadir=/usr/local/mysql/data/mysql
```

**10. 启动服务**

*Ps. 该`service`命令可用来查看服务状态(status)、停止服务(stop)等*

```
# service mysqld start
```

**11. 测试连接**

*Ps. 用户相关管理和授权等，另起一篇再记录。*

```
# ./bin/mysql -u root
```

**12. 退出`root`用户并为普通用户添加环境变量**

*Ps. `root`用户当然也可以配置，但是基本上不怎么使用`root`用户来登陆系统，所以没必要。*

```
# exit
$ vi ~/.bash_profile

PATH=$PATH:/usr/local/mysql/bin
export PATH

$ source ~/.bash_profile
```

## CentOS 开启3306端口

> CentOS 7.0默认使用的是firewall作为防火墙，使用iptables必须重新设置一下

**1. 直接关闭防火墙**

```
$ sudo systemctl stop firewalld.service         #停止firewall
$ sudo systemctl disable firewalld.service     #禁止firewall开机启动
```

**2. 安装设置`iptables service`**

```
$ sudo yum -y install iptables-services
```

然后要修改防火墙配置，比如我们现在需要新增开启3306端口，则可以编辑`iptables`配置，仿照22端口增加一条规则

```
$ sudo vi /etc/sysconfig/iptables

-A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT
```

**3. 开启防火墙，并重启系统使之生效**

```
$ sudo systemctl restart iptables.service      #重启防火墙使配置生效
$ sudo systemctl enable iptables.service      #设置防火墙开机启动
$ shutdown -r now
```

功成。

