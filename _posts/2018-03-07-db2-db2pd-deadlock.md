---
layout: post
author: "杨小定定"
title:  "DB2 使用db2pd实用工具监控/定位死锁"
description: "今天这篇继续看如何使用db2pd工具来监控或定位死锁。"
date:   2018-03-07
categories: technology
---

> 如无特别说明，均需实例用户操作

我们需要检测死锁包含两种情况：

1. 系统还未发生死锁，我们需要监控它
2. 已经疑似发生死锁，我们定位它

对于第一种情况，即开启监控，我们可以使用`db2pdcfg -catch`来开启监控。  
对于第二种情况，我们不需要监控这一步，直接分析执行`db2pd`即可。

## 开启监控

在开启之前，我们需要先准备`db2cos`脚本来处理当捕获到指定的监控内容后执行。

在DB2 v9.5以后，会优先使用`$INSTANCE/sqllib/adm/db2cos`，如找不到该脚本，则使用默认的`$INSTANCE/sqllib/bin/db2cos`；

默认情况下，`$INSTANCE/sqllib/adm/db2cos`是不存在的，所以我们要先准备一个该脚本；

同时，`db2pdcfg -catch`调用该脚本时会传入一些参数，我们也需要先接收和处理这些参数（可以从`$INSTANCE/sqllib/bin/db2cos`中复制过来），最终脚本如下：

```bash
$ cat ~/sqllib/adm/db2cos
#!/bin/bash

# In this parsing we have to recognize all the arguments. Do not want to allow
# parameters without '=' sign, or without being correctly parsed
while [ "$#" -gt "0" ]
do
   TAG=; VALUE=;
   TAG=`echo $1| sed 's/=.*//'`; VALUE=`echo $1| sed 's/'$TAG'=*//'`;
   arguments="$arguments\n $1";

   if [ ! -n "$VALUE" ]
   then
      arguments="$arguments-- Bad formed string "
   else
      case $TAG in
      # Here we recognize the parameters that are common to all automatic callout
      # scripts. These are passed by the engine
         INSTANCE)        instance=$VALUE         ;;
         DATABASE)        database=$VALUE         ;;
         DBPART)          dbpart=$VALUE           ;;
         PID)             pid=$VALUE              ;;
         TID)             tid=$VALUE              ;;
         EDUID)           eduid=$VALUE            ;;
         FUNCTION)        function=$VALUE         ;;
         COMPONENT)       component=$VALUE        ;;
         PROBE)           probe=$VALUE            ;;
         TIMESTAMP)       timestamp=$VALUE        ;;
         APPID)           appid=$VALUE            ;;
         APPHDL)          apphdl=$VALUE           ;;
         DIAGPATH)        diagpath=$VALUE         ;;
         CADIAGPATH)      ca_diagpath=$VALUE      ;;
         REASON)          reason=$VALUE           ;;
         DESCRIPTION)     description=$VALUE      ;;
         DB2_DEBUG)       db2_debug=$VALUE        ;;
         OS)              OS=$VALUE               ;;
         ISCOORD)         iscoord=$VALUE          ;;
         SDENABLED)       sdEnabled=$VALUE        ;;
         DPFENABLED)      dpfEnabled=$VALUE       ;;
         INSTALLPATH)     installPath=$VALUE      ;;
         DUMPSHM)         dumpShm=$VALUE          ;;
         DUMPDIR)         dumpDir=$VALUE          ;;
         COSTIMEOUT)      costimeout=$VALUE       ;;
         FIRSTTRAPEDU)    firstTrappedEDU=$VALUE  ;;

         *) # If it is unrecognized add a comment to the log file
            arguments="$arguments Unrecognized TAG: $TAG"
            ;;
      esac
   fi
   shift
done

echo "Lock Deadlock Caught" >> $HOME/sqllib/db2dump/db2cos.rpt
date >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Instance " $instance >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Datbase: " $database >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Parttion Number: " $dbpart >> $HOME/sqllib/db2dump/db2cos.rpt
echo "PID: " $pid >> $HOME/sqllib/db2dump/db2cos.rpt
echo "TID: " $tid >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Function: " $function >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Component: " $component >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Probe: " $probe >> $HOME/sqllib/db2dump/db2cos.rpt
echo "Timestamp: " $timestamp >> $HOME/sqllib/db2dump/db2cos.rpt
echo "AppID: " $appid >> $HOME/sqllib/db2dump/db2cos.rpt
echo "AppHdl: " $apphld >> $HOME/sqllib/db2dump/db2cos.rpt

db2pd -db $database >> $HOME/sqllib/db2dump/db2cos.rpt
```

将上述脚本添加执行权限：

```bash
$ chmod u+x ~/sqllib/adm/db2cos
```

然后就可以开启监控了，我们要监控死锁的话，可以使用下面命令两者之一，效果相同：

```bash
$ db2pdcfg -catch deadlock
$ db2pdcfg -catch -911,2

Error Catch #1
   Sqlcode:        0
   ReasonCode:     0
   ADMCode:        0
   DiagText:       
   ZRC:            -2146435070
   ECF:            0
   Component ID:   0
   LockName:       Not Set
   LockType:       Not Set
   Current Count:  0
   Max Count:      1
   Bitmap:         0x4A1
   Action:         Error code catch flag enabled
   Action:         Execute /home/db2inst1/sqllib/adm/db2cos callout script
   Action:         Produce stack trace in db2diag.log
```

然后我们可以查看`$INSTANCE/sqllib/db2dump/db2diag.log`最后输出如下字样，表示开始监控死锁了：

```bash
...
2018-01-04-13.51.16.560222+480 I928827E311           LEVEL: Event
PID     : 10731                TID : 139796437264256 PROC : db2pdcfg
INSTANCE: db2inst1             NODE : 000
HOSTNAME: study.centos.rick
FUNCTION: DB2 UDB, RAS/PD component, pdErrorCatch, probe:30
START   : Error catch set for ZRC -2146435070
```

## 制造死锁

接下来我们造一个死锁的情况，来看看具体如何分析。

先做好两个表，分别有如下数据：

```bash
$ db2 connect to TESTDB
$ db2 "select * from test_for_deadlock_1"

ID          NAME      
----------- ----------
          1 test1     

$ db2 "select * from test_for_deadlock_2"

ID          NAME      
----------- ----------
          2 test2
```

然后我们打开一个新的窗口，称之为*窗口1*：

> *使用`+c`选项，表示不自动提交；*

在其中进行一次`update`操作，并不提交：

```bash
$ db2 +c
db2 => connect to TESTDB

    Database Connection Information

  Database server         = DB2/LINUXX8664 11.1.2.2
  SQL authorization ID    = DB2INST1
  Local database alias    = TESTDB

db2 => update test_for_deadlock_1 set name='after' where id=1
DB20000I The SQL command completed successfully.
db2 =>
```

再新打开一个窗口，称之为*窗口2*：

> *使用`+c`选项，表示不提交；*

在其中进行两次`update`操作，并不提交：

```bash
$ db2 +c
db2 => connect to TESTDB

    Database Connection Information

  Database server         = DB2/LINUXX8664 11.1.2.2
  SQL authorization ID    = DB2INST1
  Local database alias    = TESTDB

db2 => update test_for_deadlock_2 set name='after' where id=2
DB20000I The SQL command completed successfully.
db2 => update test_for_deadlock_1 set name='delete' where id=1

```

此时，*窗口2*中的该命令会挂起，处于锁等待状态，等待*窗口1*中的`update`操作提交后才能继续。

然后我们切换到*窗口1*，并再执行一个`update`操作：

```bash
db2 => update test_for_deadlock_2 set name='delete' where id=2

```

此刻，该命令也同样会挂起，处于锁等待状态，等待*窗口2*中的`update`操作提交后才能继续。

至此，我们已经成功制造了一个死锁的情况。

## 回滚

等待一段时间（取决于数据库配置的参数**DLCHKTIME**的设置，默认为10秒），就会发现*窗口2*中的事务因为死锁而回滚。

```bash
SQL0911N The current transaction has been rolled back because of a deadlock
 or timeout. Reason code "2". SQLSTATE=40001
```

而*窗口1*中的命令执行成功：

```bash
db2 => update test_for_deadlock_2 set name='delete' where id=2
DB20000I The SQL command completed successfully.
db2 =>
```

*Ps.实际上也有可能是窗口1中回滚，窗口2中执行成功。*

> 数据库配置参数DLCHKTIME，配置的是毫秒数。

## 跟踪`db2diag.log`

我们可以一直开着另一个窗口跟踪`$INSTANCE/sqllib/db2dump/db2diag.log`日志。

当发生上述死锁时，我们可以看到该日志输出如下字样：

```bash
...
2018-01-04-13.55.17.406727+480 I929139E2544          LEVEL: Event
PID     : 6238                 TID : 140176680544000 PROC : db2sysc 0
INSTANCE: db2inst1             NODE : 000            DB   : TESTDB
APPHDL  : 0-27                 APPID: *LOCAL.db2inst1.180104025732
AUTHID  : DB2INST1             HOSTNAME: study.centos.rick
EDUID   : 53                   EDUNAME: db2agent (TESTDB) 0
FUNCTION: DB2 UDB, lock manager, sqlplWaitOnWP, probe:999
MESSAGE : ZRC=0x80100002=-2146435070=SQLP_LDED "Dead lock detected"
          DIA8002C A deadlock has occurred, rolling back transaction.
DATA #1 : <preformatted>
Caught rc 0x80100002.  Dumping stack trace.
CALLSTCK: (Static functions may not be resolved correctly, as they are resolved to the nearest symbol)
  [0] 0x00007F7D83DD7A7D pdLogPrintf + 0x8D
  [1] 0x00007F7D896F2392 _Z13sqlplWaitOnWPP9sqeBsuEduP14SQLP_LOCK_INFOP8SQLP_LRBP15SQLP_LTRN_CHAINbbb + 0x1AF2
  [2] 0x00007F7D896E97FD _Z24sqlplMakeNewRequestNonSDP9sqeBsuEduP14SQLP_LOCK_INFOP11SQLP_TENTRYP8SQLP_LRBS6_P15SQLP_LTRN_CHAINbbb + 0x88D
  [3] 0x00007F7D8955C997 _Z7sqlplrqP9sqeBsuEduP14SQLP_LOCK_INFO + 0xE37
  [4] 0x00007F7D844FE84F _Z12sqldReadNormP13SQLD_DFM_WORKl + 0x68F
  [5] 0x00007F7D84505B9D /home/db2inst1/sqllib/lib64/libdb2e.so.1 + 0x2011B9D
  [6] 0x00007F7D844FD890 _Z7sqldfrdP13SQLD_DFM_WORK + 0xE20
  [7] 0x00007F7D8444810D _Z12sqldRowFetchP8sqeAgentP8SQLD_CCBmmPP10SQLD_VALUEP8SQLZ_RIDmP12SQLD_ID_LISTP9SQLP_LSN8 + 0x204D
  [8] 0x00007F7D89FE4E97 _Z17sqlritaSimplePermP8sqlrr_cb + 0x567
  [9] 0x00007F7D89F00CF7 _Z15sqlriSectInvokeP8sqlrr_cbP12sqlri_opparm + 0x4B7
  [10] 0x00007F7D899306D0 _Z23sqlrr_execute_immediateP8sqlrr_cbi + 0x590
  [11] 0x00007F7D89911C6E _Z14sqlrr_execimmdP14db2UCinterfaceP16db2UCprepareInfo + 0x83E
  [12] 0x00007F7D87B0864F _Z19sqljs_ddm_excsqlimmP14db2UCinterfaceP13sqljDDMObject + 0x3EF
  [13] 0x00007F7D87AD004B _Z21sqljsParseRdbAccessedP13sqljsDrdaAsCbP13sqljDDMObjectP14db2UCinterface + 0x11B
  [14] 0x00007F7D87AD10FE _Z10sqljsParseP13sqljsDrdaAsCbP14db2UCinterfaceP8sqeAgentb + 0x54E
  [15] 0x00007F7D87AC402A /home/db2inst1/sqllib/lib64/libdb2e.so.1 + 0x55D002A
  [16] 0x00007F7D87ACA603 /home/db2inst1/sqllib/lib64/libdb2e.so.1 + 0x55D6603
  [17] 0x00007F7D87ACB2BF _Z17sqljsDrdaAsDriverP18SQLCC_INITSTRUCT_T + 0x11F
  [18] 0x00007F7D87506B53 _ZN8sqeAgent6RunEDUEv + 0xDE3
  [19] 0x00007F7D8AD1EC96 _ZN9sqzEDUObj9EDUDriverEv + 0x116
  [20] 0x00007F7D892D7358 sqloEDUEntry + 0x578
  [21] 0x00007F7D90B3EE25 /lib64/libpthread.so.0 + 0x7E25
  [22] 0x00007F7D80FB734D clone + 0x6D

2018-01-04-13.55.20.168521+480 I931684E386           LEVEL: Event
PID     : 10826                TID : 139771401488256 PROC : db2vend (PD Vendor Process - 53)
INSTANCE: db2inst1             NODE : 000
HOSTNAME: study.centos.rick
FUNCTION: DB2 UDB, trace services, pdInvokeCalloutScriptDirect, probe:10
START   : Invoking /home/db2inst1/sqllib/adm/db2cos from lock manager sqlplWaitOnWP

2018-01-04-13.55.42.900671+480 I932071E435           LEVEL: Warning
PID     : 10826                TID : 139771401488256 PROC : db2vend (PD Vendor Process - 53)
INSTANCE: db2inst1             NODE : 000
HOSTNAME: study.centos.rick
FUNCTION: DB2 UDB, trace services, pdInvokeCalloutScriptDirect, probe:130
MESSAGE : Detected end of execution of call-out script
DATA #1 : Process ID, 4 bytes
10828
DATA #2 : unsigned integer, 4 bytes
21

2018-01-04-13.55.42.901491+480 I932507E365           LEVEL: Event
PID     : 10826                TID : 139771401488256 PROC : db2vend (PD Vendor Process - 53)
INSTANCE: db2inst1             NODE : 000
HOSTNAME: study.centos.rick
FUNCTION: DB2 UDB, trace services, pdInvokeCalloutScriptDirect, probe:150
STOP    : Completed invoking /home/db2inst1/sqllib/adm/db2cos

2018-01-04-13.55.42.903050+480 I932873E520           LEVEL: Warning
PID     : 6238                 TID : 140176680544000 PROC : db2sysc 0
INSTANCE: db2inst1             NODE : 000            DB   : TESTDB
APPHDL  : 0-27                 APPID: *LOCAL.db2inst1.180104025732
AUTHID  : DB2INST1             HOSTNAME: study.centos.rick
EDUID   : 53                   EDUNAME: db2agent (TESTDB) 0
FUNCTION: DB2 UDB, trace services, pdInvokeCalloutScriptViaVendorAPI, probe:16
DATA #1 : <preformatted>
PD Vendor Return Code: 0
```

上面日志提示捕获到死锁发生，并开始执行以及结束执行`/home/db2inst1/sqllib/adm/db2cos`。

## 分析`db2pd`输出

我们通过准备的`db2cos`最后一行可以看出：

当执行的时候，实际上是执行`db2pd -db $database`来获取数据库所有的`db2pd`的输出。

*Ps.当然，我们也可以修改它来只获取部分输出。*

我们查看输出文件`$HOME/sqllib/db2dump/db2cos.rpt`：

**第一步：**先找到`Locks:`块，锁的信息

```bash
Locks:
Address            TranHdl    Lockname                   Type           Mode Sts Owner      Dur HoldCount  Att        ReleaseFlg rrIID
0x00007F7D189B2600 12         040000000100000001006051D6 VarLock        ..S  G   12         1   0          0x00000000 0x40000000 0
0x00007F7D189B4A00 3          040000000100000001008047D6 VarLock        ..S  G   12         1   0          0x00000000 0x40000000 0
0x00007F7D189B4E80 3          02000800040000000000000052 RowLock        ..X  G   12         1   0          0x00200000 0x40000000 0
0x00007F7D189B2E00 12         02000800040000000000000052 RowLock        ..U  W*  12         0   0          0x00004000 0x00000000 0
0x00007F7D189B2A00 12         02000900040000000000000052 RowLock        ..X  G   12         1   0          0x00200000 0x40000000 0
0x00007F7D189B2480 3          02000900040000000000000052 RowLock        ..U  W   12         0   0          0x00000000 0x00000000 0
0x00007F7D189BDC80 3          41414141416641647CF81EA6C1 PlanLock       ..S  G   12         1   0          0x00000000 0x40000000 0
0x00007F7D189B2F00 12         41414141416641647CF81EA6C1 PlanLock       ..S  G   12         1   0          0x00000000 0x40000000 0
0x00007F7D189B2780 3          02000800000000000000000054 TableLock      .IX  G   12         1   0          0x00202000 0x40000000 0
0x00007F7D189B2E80 12         02000800000000000000000054 TableLock      .IX  G   12         1   0          0x00203000 0x40000000 0
...
```

注意看其中状态`Sts`为`W*`的锁，这个就是最后**回滚**的那个应用程序的锁。

上面的锁信息为

```bash
0x00007F7D189B2E00 12         02000800040000000000000052 RowLock        ..U  W*  12         0   0          0x00004000 0x00000000 0
```

我们看到事务句柄`TranHdl为12`，同时看到持有这个锁`02000800040000000000000052`的另一个应用程序的句柄为3。

```bash
0x00007F7D189B4E80 3          02000800040000000000000052 RowLock        ..X  G   12         1   0          0x00200000 0x40000000 0
```

**第二步：**找到`Transactions:`块，事务信息

```bash
Transactions:
Address            AppHandl [nod-index] TranHdl    Locks      State   Tflag      Tflag2     Firstlsn           Lastlsn            Firstlso             Lastlso              LogSpace        SpaceReserved   TID            AxRegCnt   GXID     ClientUserID                   ClientWrkstnName               ClientApplName                 ClientAccntng
0x00007F7D18768F00 271      [000-00271] 3          6          WRITE   0x00000000 0x00000000 0x00000000000972D2 0x00000000000972D2 137069032            137069032            197             292             0x0000000072AB 1          0        n/a                                 n/a                            n/a                            n/a
0x00007F7D1876BE80 9        [000-00009] 4          0          READ    0x00000000 0x00000000 0x0000000000000000 0x0000000000000000 0                    0                    0               0               0x000000006CF3 1          0        n/a                                 n/a                            n/a                            n/a
...
0x00007F7D18783A80 272      [000-00272] 12         6          WRITE   0x00000000 0x00000000 0x00000000000972D3 0x00000000000972D3 137069127            137069127            197             292             0x0000000072AA 1          0        n/a                                 n/a                            n/a                            n/a
0x00007F7D18786A00 18       [000-00018] 13         0          READ    0x00000000 0x00000000 0x0000000000000000 0x0000000000000000 0                    0                    0               0               0x000000006CFF 1          0        n/a                                 n/a                            db2evmg_DB2DETAILDEADLOCK      n/a
```

我们可以看到事务句柄`TranHdl为12`对应的应用程序句柄`AppHandl为272`;

事务句柄`TranHdl`为3对应的应用程序句柄`AppHandl为271`。

**第三步：**找到`Applications:`块，应用程序信息

```bash
Applications:
Address            AppHandl [nod-index] NumAgents  CoorEDUID  Status                  C-AnchID C-StmtUID  L-AnchID L-StmtUID  Appid                                                            WorkloadID  WorkloadOccID CollectActData          CollectActPartition     CollectSectionActuals
...
0x0000000203BDEBC0 272      [000-00272] 1          53         UOW-Executing           651      4          236      4          *LOCAL.db2inst1.180104061332                                     1           4             N                       C                       N
0x0000000203B30080 15       [000-00015] 1          48         ConnectCompleted        0        0          0        0          *LOCAL.DB2.180104025535                                          0           0             N                       C                       N
0x0000000203B61720 271      [000-00271] 1          22         Lock-wait               572      4          280      4          *LOCAL.db2inst1.180104061319                                     1           3             N                       C                       N
...
```

我们可以看到应用程序句柄`AppHandl`为`272`以及`271`的记录。

其中的*当前运行锚ID*为`C-AnchID`、*当前运行语句UID*为`C-StmtUID`；*最近锚ID*为`L-AnchID`、*最近语句UID*`L-StmtUID`。

在此例中，`272`对应的是当前语句是`651`、`4`；

`271`对应的当前语句是`572`、`4`。

**第四步：**找到`Dynamic SQL Statements:`块，当前正在执行的SQL语句

```bash
Dynamic SQL Statements:
Address            AnchID StmtUID    NumEnv     NumVar     NumRef     NumExe     Text
0x00007F7D21BAC960 129    2          0          0          1          1          select * from test_for_deadlock_2
...
0x00007F7D21BAC760 559    2          0          0          1          1          select * from test_for_deadlock_1
0x00007F7D1D0BF8C0 572    4          1          1          1          1          update test_for_deadlock_2 set name='delete' where id=2
0x00007F7D21BA9AA0 651    4          1          1          1          1          update test_for_deadlock_1 set name='delete' where id=1
...
```

我们就可以找到`AnchID`和`StmtUID`分别为`651`、`4`和`572`、`4`对应的语句了。

**第五步：**进行业务分析

上面就是我们通过`db2pd`实用程序来监控/定位死锁。
一旦我们定位引发死锁的语句，就可以根据业务逻辑对该SQL语句进行条有和创建合理的索引。
