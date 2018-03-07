---
layout: post
author: "杨小定定"
title:  "DB2 for Linux 安装"
description: "记录在CentOS 7系统下安装DB2社区版过程。"
date:   2018-03-05 14:05:00 +0800
categories: technology
---

我们通过IBM官网，下载DB2 Developer Community Edition v11.1。

> 注意：暂且不讨论基于docker容器版，我们仍然下载普通版（v11.1_linuxx64_dec.tar.gz）。

下载完成后，解压，得到一个目录，目录下文件如下：

```bash
$ tar -zxvf v11.1_linuxx64_dec.tar.gz
$ cd server_dec
$ ls
db2  db2ckupgrade  db2_deinstall  db2_install  db2ls  db2prereqcheck  db2setup  installFixPack  nlpack
```

我们可以看到上述包含有`db2_install`和`db2setup`两个执行文件，即有两种安装方式：

1. 使用`db2_install`采用命令行模式安装
2. 使用`db2setup`采用界面安装

下面我们就来分别说说这两种方式。

## 命令安装

通过命令安装，我们打开终端，切换`root`用户，将该目录复制到`/home/software`（当然，其它地方也行...），如下：

```
# pwd
/home/software/server_dec
# ls
db2  db2ckupgrade  db2_deinstall  db2_install  db2ls  db2prereqcheck  db2setup  installFixPack  nlpack
```

### 第一步：检查环境

首先需要通过DB2的工具来检查安装环境是否符合要求，此工具即当前目录下的`db2prereqcheck`，可以通过`-v`来指定具体检查的版本，此处我们安装的是`v11.1.2.2`，所以我们指定它：

```bash
# ./db2prereqcheck -v 11.1.2.2
...
```

上面检查输出显示缺少哪些先决条件，就先安装即可。

### 第二步：执行安装

执行安装，直接运行`db2_install`即可，它会以交互模式来运行。

先要求同意协议，输入yes回车；  
然后提示是否安装至缺省目录 (`/opt/ibm/db2/V11.1`)？输入yes回车；  
再选择产品，提示：

```bash
指定下列其中一个关键字以安装 DB2 产品。

  SERVER 
  CLIENT 
  RTCL
```

我们输入`SERVER`回车。

然后就会继续安装了，很简单，我们等它安装完毕即可。

*Ps.在我们输入完SERVER后，它会提示要安装 DB2 pureScale Feature 吗？*  
*因我们在执行`db2prereqcheck`时，它已经明确提示了，当前操作系统（CentOS7）不支持它，所以此处我们选择不安装。*

直到看到如下提示，即表示安装完毕：

```bash
...

已成功完成执行。

有关更多信息，请参阅 "/tmp/db2_install.log.9382" 上的 DB2
安装日志。
#
```

### 第三步：创建用户

接着我们来创建三个用户和用户组，分别如下：

```bash
# groupadd db2iadm1
# groupadd db2fadm1
# groupadd dasadm1
# useradd -m -g db2iadm1 -d /home/db2inst1 db2inst1
# useradd -m -g db2fadm1 -d /home/db2fenc1 db2fenc1
# useradd -m -g dasadm1 -d /home/dasusr1 dasusr1
```

*这三个用户分别是什么作用就不同在这里说了吧...*

### 第四步：创建实例

至此，我们还只是安装了DB2数据库，以及手工创建了准实例用户。还没有实例存在，所以我们来创建一个实例。

进入DB2安装目录，使用`db2icrt`来创建实例：

```bash
# cd /opt/ibm/db2/V11.1/instance
# ./db2icrt -u db2fenc1 db2inst1
...
```

直到提示实例创建成功。

我们如果想查看所有实例，可以在`bin`目录下运行`db2ilist`工具，如：

```bash
# cd /opt/ibm/db2/V11.1/bin
# ./db2ilist
db2inst1
```

如果想查看当前实例，可以使用`db2 get instance`命令，如：

```bash
# su - db2inst1
$ db2 get instance

 当前数据库管理器实例是：db2inst1
```

若想删除实例，也很简单，使用`bin`目录下的`db2idrop`工具即可，如：

*Ps.需要root用户*

```bash
# cd /opt/ibm/db2/V11.1/bin
# ./db2idrop db2inst1
```

*再Ps.可以使用`-f`参数强制删除*

### 第五步：安装license

我们解压的安装包里面，就包含有`license`文件，如下：

```bash
# cd /home/software/server_dec/db2/license
# ls | grep db2
db2dec.lic    db2ese.lic
```

我们安装的是`dec`版，使用`db2dec.lic`即可。

查看是否注册，使用`db2licm -l`命令，如：

```bash
# su - db2inst1
$ db2licm -l
产品名：                          "DB2 Enterprise Server Edition"
许可证类型：                     "许可证未注册"
到期日期：                        "许可证未注册"
产品标识：                       "db2ese"
版本信息：                        "11.1"
```

接下来我们执行注册：

```bash
$ db2licm -a /home/software/server_dec/db2/license/db2dec.lic

LIC1402I  成功地添加了许可证。


LIC1426I  现在，您已获取本产品的使用许可，详细信息请参阅许可协议。 使用产品表示接受位于以下目录中的 IBM 许可协议中的条款： "/opt/ibm/db2/V11.1/license/zh_CN.utf8"
```

再来查看注册信息：

```bash
$ db2licm -l
产品名：                          "DB2 Enterprise Server Edition"
许可证类型：                     "许可证未注册"
到期日期：                        "许可证未注册"
产品标识：                       "db2ese"
版本信息：                        "11.1"

产品名：                          "IBM DB2 Developer-C Edition"
许可证类型：                     "Community"
到期日期：                        "永久"
产品标识：                       "db2dec"
版本信息：                        "11.1"
最大内存量 (GB)：               "16"
最大核心数：              "4"
最大表空间 (GB)：   "100"
```

即注册完成。

### 第六步：启动实例

启动实例很简单了，切换到实例用户，运行·db2start·即可，如：

```bash
$ db2start
12/14/2017 11:03:34     0   0   SQL1063N  DB2START 处理成功。
SQL1063N  DB2START 处理成功。
```

若想停止实例，运行`db2stop`即可，如：

```bash
$ db2stop
2017-12-14 11:05:35     0   0   SQL1064N  DB2STOP 处理成功。
SQL1064N  DB2STOP 处理成功。
```

*Ps.可使用db2stop force来强制停止实例。*

### 第七步：设定监听端口

当启动DB2实例时，若发现，DB2的监听端口并没有随着DB2实例的启动而启动。

那么下面过程设置如何启动监听。

先执行`db2set-all`来检查是否有`DB2COMM=TCPIP`一项，如果没有则应该执行`db2set DB2COMM=TCPIP`设置：

```bash
$ db2set –all
$ db2set DB2COMM=TCPIP
```

再来检查`SVCENAME`的值：

```bash
$ db2 get dbm cfg | grep SVCENAME
```

如果`SVCENAME`为空值，则需要用下面的步骤设定该值；

如果是一个端口号（端口号应小于65536），则不用读取`/etc/services`文件中的端口定义；  
如果该值是一个字符串（如：`db2c_db2inst1`），则在实例启动时会自动读取`/etc/services`中的该字符串对应的端口号来监听。

```bash
$ db2 update database manager configuration using svcename db2c_db2inst1
```

*Ps.其中`db2c_db2inst1`也可以为端口号如：`50001`，我们也可以通过这种方式修改DB2监听的端口号。*

### 第八步：DB2管理服务器

首先创建DB2管理服务器：

```bash
# cd /opt/ibm/db2/V9.5/instance
# ./dascrt -u dasusr1
SQL4406W The DB2 Administration Server was started successfully.
DBI1070I Program dascrt completed successfully.
```

再启动DB2管理服务器：

```bash
# su - dasusr1
# db2admin start
SQL4409W The DB2 Administration Server is already active.
```

启动完成，这时可以用命令`netstat-an`查看DB2管理服务器的监听端口523是否被监听。

*说明：DB2管理服务器启动完成后，可以通过客户端对服务器数据库进行管理，比如在windows机器上通过DB2控制中心访问远端服务器数据库！*

若要停止DB2管理服务器，通过管理服务器用户执行`db2admin stop`即可，如：

```bash
# su – dasusr1
$ db2admin stop
```

至此，通过命令方式安装与配置DB2数据库完成。

## 界面安装

### 第一步：启动安装

通过界面安装就很简单了，直接执行`db2setup`，它会启动安装界面：

```bash
# cd /home/software/server_dec
# ./db2setup
```

我们根据界面提示，选择全新安装，设置好用户密码，一路下一步即可。

*Ps.它会自动帮我们创建实例用户、防护用户、以及实例。*

最后安装完成。

### 第二步：安装license

和使用命令安装一样，过程就不赘述了，参考上面。

就完成了。

## 创建SAMPLE数据库

经过上面安装数据库，无论是命令行安装的，还是界面安装的，都只完成了安装数据库软件，以及创建实例。

DB2还自带了一个SAMPLE库，我们可以通过命令`db2sampl`来创建，会包含一些示例的表和数据：

```bash
# su - db2inst1
$ db2sampl
...
```

完成后我们就可以连接该库，进行DB2学习与操作了：

```bash
$ db2 connect to sample

   数据库连接信息

 数据库服务器         = DB2/LINUXX8664 11.1.2.2
 SQL 授权标识         = DB2INST1
 本地数据库别名       = SAMPLE
```
