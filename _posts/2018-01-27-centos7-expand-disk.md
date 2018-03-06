---
layout: post
author: "杨小定定"
title:  "CentOS 7 扩展磁盘操作"
description: "本文记录了在WMWare中将已经安装好的CentOS 7系统扩展磁盘的操作"
date:   2018-01-27 23:23:01 +0800
categories: Linux CentOS
---

本次是记录在VMWare中安装了CentOS 7系统，本来预先分配了40G的磁盘，其中空间分配如下：

```
[rick@study ~]$df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   10G  5.0G  5.1G  50% /
devtmpfs                 901M     0  901M   0% /dev
tmpfs                    916M   96K  916M   1% /dev/shm
tmpfs                    916M  9.0M  907M   1% /run
tmpfs                    916M     0  916M   0% /sys/fs/cgroup
/dev/sda2               1014M  173M  842M  18% /boot
/dev/mapper/centos-home  5.0G  473M  4.6G  10% /home
tmpfs                    184M  4.0K  184M   1% /run/user/42
tmpfs                    184M  8.0K  184M   1% /run/user/1000
```

因需要安装DB2，安装过程中报磁盘空间不足，故需要扩容。

## VMWare操作

首先关掉虚拟机，在 虚拟机-->设置-->硬盘-->实用工具-->扩容，调整硬盘的大小。

*此时要注意，如果之前有做过虚拟机快照的，需要将快照删除才能调整。*

然后扩容完成即可。

启动虚拟机，进入CentOS。

## 查看分区

切换到`root`用户。

一、查看分区情况

运行`lsblk`可以查看磁盘列表，如下：

```
[root@study ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0               2:0    1    4K  0 disk 
sda               8:0    0   60G  0 disk 
├─sda1            8:1    0    2M  0 part 
├─sda2            8:2    0    1G  0 part /boot
├─sda3            8:3    0   30G  0 part 
│ ├─centos-root 253:0    0   10G  0 lvm  /
│ ├─centos-swap 253:1    0    1G  0 lvm  [SWAP]
│ └─centos-home 253:2    0    5G  0 lvm  /home
├─sda4            8:4    0    1G  0 part 
└─sda5            8:5    0    1G  0 part 
sr0              11:0    1 1024M  0 rom  
```

我们可以看到，硬盘主要有`fd0`和`sda`两块，因为我们已经在VMWare中将虚拟硬盘扩容了，故此处显示`sda`大小有60G。

其中，`sda`硬盘上有`sda1`~`sda5`五个分区，`sda3`上又有3个逻辑分区。

*Ps.`sda4`和`sda5`这两个分区是之前测试时使用的，此处不管。*

很明显可以看到`sda`这块硬盘上还有剩余空间，我们继续。

## 创建新的分区

所以我们对`sda`进行分割操作，这个操作的话，有`fdisk`和`gdisk`两个命令。

这两个命令的操作都差不多，但是适用对象不一样。我们首先利用`parted /dev/sda print`来查看一下：

```
[root@study ~]# parted /dev/sda print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 64.4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: pmbr_boot

Number  Start   End     Size    File system  Name                  标志
 1      1049kB  3146kB  2097kB                                     bios_grub
 2      3146kB  1077MB  1074MB  xfs
 3      1077MB  33.3GB  32.2GB                                     lvm
 4      33.3GB  34.4GB  1074MB  xfs          Linux filesystem
 5      34.4GB  35.4GB  1074MB  ext4         Microsoft basic data
```

注意到上面结果第4行，分区表为`gpt`，所以就要使用`gdisk`命令；如果是`mbr`则使用`fdisk`。

所以此处我们使用`gdisk`来分割磁盘，创建两个新的分区`sda6`和`sda7`。

先查看一下磁盘信息：

```
[root@study ~]# gdisk /dev/sda -l
GPT fdisk (gdisk) version 0.8.6

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.
Disk /dev/sda: 125829120 sectors, 60.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 8CCC7A48-02D8-4295-91C3-912957F08A6A
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 125829086
Partitions will be aligned on 2048-sector boundaries
Total free space is 56610749 sectors (27.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            6143   2.0 MiB     EF02  
   2            6144         2103295   1024.0 MiB  0700  
   3         2103296        65026047   30.0 GiB    8E00  
   4        65026048        67123199   1024.0 MiB  8300  Linux filesystem
   5        67123200        69220351   1024.0 MiB  0700  Microsoft basic data
```

上面结果主要关注的有：

- `Total free space is 56610749 sectors (27.0 GiB)`显示还有多少可用
- 最后一个的`End (sector)`，即最后一块地址

我们接下来正式开始分割`/dev/sda`，如下：

```
[root@study ~]# gdisk /dev/sda
GPT fdisk (gdisk) version 0.8.6

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): ?
b    back up GPT data to a file
c    change a partition's name
d    delete a partition
i    show detailed information on a partition
l    list known partition types
n    add a new partition
o    create a new empty GUID partition table (GPT)
p    print the partition table
q    quit without saving changes
r    recovery and transformation options (experts only)
s    sort partitions
t    change a partition's type code
v    verify disk
w    write table to disk and exit
x    extra functionality (experts only)
?    print this menu

Command (? for help): n
Partition number (6-128, default 6): 
First sector (34-125829086, default = 69220352) or {+-}size{KMGTP}: 
Last sector (69220352-125829086, default = 125829086) or {+-}size{KMGTP}: +5G
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sda: 125829120 sectors, 60.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 8CCC7A48-02D8-4295-91C3-912957F08A6A
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 125829086
Partitions will be aligned on 2048-sector boundaries
Total free space is 46124989 sectors (22.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            6143   2.0 MiB     EF02  
   2            6144         2103295   1024.0 MiB  0700  
   3         2103296        65026047   30.0 GiB    8E00  
   4        65026048        67123199   1024.0 MiB  8300  Linux filesystem
   5        67123200        69220351   1024.0 MiB  0700  Microsoft basic data
   6        69220352        79706111   5.0 GiB     8300  Linux filesystem

Command (? for help): n
Partition number (7-128, default 7): 
First sector (34-125829086, default = 79706112) or {+-}size{KMGTP}: 
Last sector (79706112-125829086, default = 125829086) or {+-}size{KMGTP}: +10G
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sda: 125829120 sectors, 60.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 8CCC7A48-02D8-4295-91C3-912957F08A6A
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 125829086
Partitions will be aligned on 2048-sector boundaries
Total free space is 25153469 sectors (12.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            6143   2.0 MiB     EF02  
   2            6144         2103295   1024.0 MiB  0700  
   3         2103296        65026047   30.0 GiB    8E00  
   4        65026048        67123199   1024.0 MiB  8300  Linux filesystem
   5        67123200        69220351   1024.0 MiB  0700  Microsoft basic data
   6        69220352        79706111   5.0 GiB     8300  Linux filesystem
   7        79706112       100677631   10.0 GiB    8300  Linux filesystem

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sda.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot.
The operation has completed successfully.
```

上述操作中，主要使用了`n`子命令，来新增分区，以及`p`子命令来查看。
其中新增分区大小，`First sector`直接默认即可，他会从最近的接着来；`Last sector`采用`+10G`这种方式来指定大小。

最后`w`保存退出即可。

## 生效新增的分区

我们接着运行`lsblk`命令，会发现刚刚新增的分区竟然没有出现：

```
[root@study ~]# lsblk /dev/sda
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda               8:0    0  60G  0 disk 
├─sda1            8:1    0   2M  0 part 
├─sda2            8:2    0   1G  0 part /boot
├─sda3            8:3    0  30G  0 part 
│ ├─centos-root 253:0    0  10G  0 lvm  /
│ ├─centos-swap 253:1    0   1G  0 lvm  [SWAP]
│ └─centos-home 253:2    0   5G  0 lvm  /home
├─sda4            8:4    0   1G  0 part 
└─sda5            8:5    0   1G  0 part 
```

这是因为系统还没有更新核心的分区表信息，有两种解决方法：

- 重启系统
- 使用`partprobe`命令更新分区表信息

重启的话太麻烦，所以我们采用第二种方法来使新增的分区生效：

```
[root@study ~]# partprobe -s
/dev/sda: gpt partitions 1 2 3 4 5 6 7
```

再次查看就有了：

```
[root@study ~]# lsblk /dev/sda
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda               8:0    0  60G  0 disk 
├─sda1            8:1    0   2M  0 part 
├─sda2            8:2    0   1G  0 part /boot
├─sda3            8:3    0  30G  0 part 
│ ├─centos-root 253:0    0  10G  0 lvm  /
│ ├─centos-swap 253:1    0   1G  0 lvm  [SWAP]
│ └─centos-home 253:2    0   5G  0 lvm  /home
├─sda4            8:4    0   1G  0 part 
├─sda5            8:5    0   1G  0 part 
├─sda6            8:6    0   5G  0 part 
└─sda7            8:7    0  10G  0 part 
```

## 格式化分区

格式化分区，其实就是创建文件系统，我们此处采用的是`xfs`格式，如：

```
[root@study ~]# mkfs.xfs /dev/sda6
meta-data=/dev/sda6              isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@study ~]# blkid /dev/sda6
/dev/sda6: UUID="be16c01e-2683-4814-b12a-e91a94011aae" TYPE="xfs" PARTLABEL="Linux filesystem" PARTUUID="6eb10297-eda3-4237-bd58-97c11522b624" 
```

接下来就可以开始添加了。

## 创建物理卷

先使用`pvdisplay`命令查看所有的物理卷（pv: physical volumn）：

```
[root@study ~]# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               centos
  PV Size               30.00 GiB / not usable 4.00 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              7680
  Free PE               3584
  Allocated PE          4096
  PV UUID               CRbau6-yCis-W2Uh-CI7T-URMz-cVqd-yr0wvG
[root@study ~]# 
```

其中，`/dev/sda3`就是物理卷名称，`centos`是物理卷组名称。

再使用`vgdisplay`命令查看所有的物理卷组（vg: volumn group）：

```
[root@study ~]# vgdisplay
  --- Volume group ---
  VG Name               centos
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                3
  Open LV               3
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               35.00 GiB
  PE Size               4.00 MiB
  Total PE              8959
  Alloc PE / Size       7680 / 30.00 GiB
  Free  PE / Size       1279 / 5.00 GiB
  VG UUID               S9p8yK-Wzxz-Qm76-1s2q-y3KF-Evdb-jaE9h5
[root@study ~]# 
```

上面结果表明，我们有一个`/dev/sda3`的物理卷，以及一个`centos`的物理卷组。

我们创建新的`/dev/sda6`和`/dev/sda7`物理卷：

```
[root@study ~]# pvcreate /dev/sda6
WARNING: xfs signature detected on /dev/sda6 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/sda6.
  Physical volume "/dev/sda6" successfully created.
[root@study ~]# pvcreate /dev/sda7
  Physical volume "/dev/sda7" successfully created.
```

*此处有一个疑问，就是对于`sda7`我是忘了进行`mkfs.xfs`格式化了。*
*但是进行`pvcreate /dev/sda6`时提示需要擦除xfs签名，而`sda7`时不用。*
*是否不需要进行创建文件系统这一步？*

此时再使用`pvdisplay`查看，就能看到多了两个物理卷了。

## 将新建的物理卷添加到卷组

需要将物理卷添加到卷组，我们使用`vgextend`命令，如下：

```
[root@study ~]# vgextend centos /dev/sda6
  Volume group "centos" successfully extended
[root@study ~]# vgextend centos /dev/sda7
  Volume group "centos" successfully extended
```

此时通过`pvdisplay`查看物理卷的话，可以看到`/dev/sda6`和`/dev/sda7`的VG均为`centos`了：

```
[root@study ~]# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               centos
  PV Size               30.00 GiB / not usable 4.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              7680
  Free PE               0
  Allocated PE          7680
  PV UUID               CRbau6-yCis-W2Uh-CI7T-URMz-cVqd-yr0wvG
   
  --- Physical volume ---
  PV Name               /dev/sda6
  VG Name               centos
  PV Size               5.00 GiB / not usable 4.00 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              1279
  Free PE               1279
  Allocated PE          0
  PV UUID               N5SXiJ-IfJb-yjNY-zggY-zpSF-FkoB-tzC7C3
   
  --- Physical volume ---
  PV Name               /dev/sda7
  VG Name               centos
  PV Size               10.00 GiB / not usable 4.00 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              2559
  Free PE               2559
  Allocated PE          0
  PV UUID               VazJqC-dHfe-uDhQ-GQPS-EJQI-gofn-A2srvI
```

我们还可以通过`vgdisplay`查看`centos`这个卷组的空闲空间：

```
[root@study ~]# vgdisplay
  --- Volume group ---
  VG Name               centos
  System ID             
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  7
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                3
  Open LV               3
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               44.99 GiB
  PE Size               4.00 MiB
  Total PE              11518
  Alloc PE / Size       7680 / 30.00 GiB
  Free  PE / Size       3838 / 14.99 GiB
  VG UUID               S9p8yK-Wzxz-Qm76-1s2q-y3KF-Evdb-jaE9h5
```

上述结果中倒数第二行的`Free`显示还有15G，即我们新增的`/dev/sda6`的5G，和`/dev/sda7`的10G。

## 将逻辑卷扩展

先可以通过`lvdisplay`查看所有的逻辑卷：

```
[root@study ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/centos/root
  LV Name                root
  VG Name                centos
  LV UUID                GUJDUV-FESa-VLFS-bIhm-b8aE-sdm2-nvP907
  LV Write Access        read/write
  LV Creation host, time study.centos.rick, 2017-01-17 19:40:45 +0800
  LV Status              available
  # open                 1
  LV Size                24.00 GiB
  Current LE             6144
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
   
  --- Logical volume ---
  LV Path                /dev/centos/home
  LV Name                home
  VG Name                centos
  LV UUID                fV6UG2-pW6N-Vow9-ZiDT-Hmb5-HbHq-8FcA3b
  LV Write Access        read/write
  LV Creation host, time study.centos.rick, 2017-01-17 19:40:46 +0800
  LV Status              available
  # open                 1
  LV Size                5.00 GiB
  Current LE             1280
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2
   
  --- Logical volume ---
  LV Path                /dev/centos/swap
  LV Name                swap
  VG Name                centos
  LV UUID                v9nATg-zM8x-MJ5c-tuTm-9NHk-7Ba7-AtYTFq
  LV Write Access        read/write
  LV Creation host, time study.centos.rick, 2017-01-17 19:40:47 +0800
  LV Status              available
  # open                 2
  LV Size                1.00 GiB
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:1
```

我们现在要将`/dev/centos/home`扩容10G，可以使用`lvextend`命令，如：

```
[root@study ~]# lvextend /dev/centos/home /dev/sda7
  Size of logical volume centos/home changed from 5.00 GiB (1280 extents) to 15.00 GiB (3839 extents).
  Logical volume centos/home successfully resized.
```

*Ps.上述`lvextend`命令后面要加物理卷路径，否则会将该逻辑卷中所有空闲的空间都扩展上。*

对于`/dev/centos/root`也类似操作，显示成功后，此时通过`df -h`查看：

```
[root@study ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   24G  5.0G   20G   21% /
devtmpfs                 901M     0  901M    0% /dev
tmpfs                    916M  152K  916M    1% /dev/shm
tmpfs                    916M  9.1M  907M    1% /run
tmpfs                    916M     0  916M    0% /sys/fs/cgroup
/dev/sda2               1014M  173M  842M   18% /boot
/dev/mapper/centos-home  5.0G  473M  4.6G   10% /home
tmpfs                    184M  4.0K  184M    1% /run/user/42
tmpfs                    184M   12K  184M    1% /run/user/1000
tmpfs                    184M     0  184M    0% /run/user/0
```

发现`/dev/mapper/centos-home`还是5G，这是因为还未刷新逻辑卷的容量，我们需要执行下面命令刷新：

```
[root@study ~]# xfs_growfs /dev/centos/home
meta-data=/dev/mapper/centos-home isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1310720 to 3931136
```

*此处因为是`xfs`格式，所以使用`xfs_growfs`命令，如果是`ext4`格式，则要使用`resize2fs`命令。*

然后再查看就有了：

```
[root@study ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   24G  5.0G   20G   21% /
devtmpfs                 901M     0  901M    0% /dev
tmpfs                    916M  152K  916M    1% /dev/shm
tmpfs                    916M  9.1M  907M    1% /run
tmpfs                    916M     0  916M    0% /sys/fs/cgroup
/dev/sda2               1014M  173M  842M   18% /boot
/dev/mapper/centos-home   15G  473M   15G    4% /home
tmpfs                    184M  4.0K  184M    1% /run/user/42
tmpfs                    184M   12K  184M    1% /run/user/1000
tmpfs                    184M     0  184M    0% /run/user/0
```

结果可以看到`/`根目录已经从最开始的5G扩展至了24G，`/home`已经从5G扩展至了15G。

*Ps.本意是想给`/`新增5G，给`/home`新增10G，结果过程中发现`/dev/sda3`分区中本来就还有14G左右的未使用空间，就一并分给`/`了。*

功成。
