---
layout: post
title: Linux LVM 逻辑卷扩容
date: 2019-08-09 11:00:00
updated: 2019-08-09 14:13:08
categories: 教程
tags: 
    - Linux
    - CentOS
    - LVM
urlname: 22
comment: true
---

接上文 [真机安装 CentOS 7](https://blackyau.cc/10.html) 在安装完毕后，我又给它加了一次硬盘。得益于安装的时候使用了 LVM (Logical Volume Manager) 可以实现不关机扩展当前分区。

<!-- more -->

## LVM 介绍

![Lvm](https://st.blackyau.net/blog/22/Lvm.svg)

> 图片来自 [Wikimedia Logical Volume Manager (Linux)](https://commons.wikimedia.org/wiki/File:Lvm.svg#/media/File:Lvm.svg)

 - PV：物理卷，PV处于LVM系统最低层，它可以是整个硬盘，或者与磁盘分区具有相同功能的设备（如RAID），但和基本的物理存储介质相比较，多了与LVM相关管理参数
 - VG：卷组，创建在PV之上，由一个或多个PV组成，可以在VG上创建一个或多个“LVM分区”（逻辑卷），功能类似非LVM系统的物理硬盘
 - LV：逻辑卷，从VG中分割出的一块空间，创建之后其大小可以伸缩，在LV上可以创建文件系统（如/var,/home）
 - PE：物理区域，每一个PV被划分为基本单元（也被称为PE），具有唯一编号的PE是可以被LVM寻址的最小存储单元，默认为4MB

本次扩容就是将设备转换为`物理卷`，然后将其加入`卷组`，最后再将`卷组`的剩余空间划分给`逻辑卷`

## 安装硬盘

这一步没啥可说的了吧，直接把硬盘接上主板就行了

## 系统环境

| 设备名 | 大小 | 状态 | 所属卷组名 |
| ----- | ---- | ---- | ---- |
| /dev/sdc | 500.1 GB | 新加盘 | 无 |
| /dev/sdb | 3000.6 GB | 原有数据盘 | all |
| /dev/sda | 500.1 GB | 原有数据盘 | all |
| /dev/sdd | 320.1 GB | 系统盘+数据盘 | centos all |

这里我会将 `/dev/sdc` 划分进 `all` 卷组中以扩大它的可用容量

查询磁盘空间信息

```shell
df -H
```

```shell
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   54G  2.1G   52G   4% /
devtmpfs                 889M     0  889M   0% /dev
tmpfs                    902M     0  902M   0% /dev/shm
tmpfs                    902M   18M  884M   2% /run
tmpfs                    902M     0  902M   0% /sys/fs/cgroup
/dev/mapper/all-home     3.8T  3.6T  252G  94% /home
/dev/sdd1                1.1G  143M  811M  15% /boot
tmpfs                    181M     0  181M   0% /run/user/1000
```

查询物理卷信息

```shell
sudo pvs
```

```shell
  PV         VG     Fmt  Attr PSize    PFree
  /dev/sda1  all    lvm2 a--  <465.76g    0 
  /dev/sdb1  all    lvm2 a--    <2.73t    0 
  /dev/sdd2  all    lvm2 a--  <245.08g    0 
  /dev/sdd3  centos lvm2 a--    52.00g 4.00m
```

查询硬盘分区表情况

```shell
sudo fdisk -l
```

```shell
Disk /dev/sdc: 500.1 GB, 500107862016 bytes, 976773168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: dos
Disk identifier: 0x95fe95fe

   Device Boot      Start         End      Blocks   Id  System
/dev/sdc1   *        4096   976773119   488384512    7  HPFS/NTFS/exFAT

Disk /dev/sdd: 320.1 GB, 320072933376 bytes, 625142448 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00084660

   Device Boot      Start         End      Blocks   Id  System
/dev/sdd1   *        2048     2099199     1048576   83  Linux
/dev/sdd2         2099200   516073471   256987136   8e  Linux LVM
/dev/sdd3       516073472   625141759    54534144   8e  Linux LVM
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.

Disk /dev/sdb: 3000.6 GB, 3000592982016 bytes, 5860533168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: gpt
Disk identifier: BDF3367A-EECA-4A93-BE0B-10D867E87AF1


#         Start          End    Size  Type            Name
 1         2048   5860532223    2.7T  Linux LVM       

Disk /dev/sda: 500.1 GB, 500107862016 bytes, 976773168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: dos
Disk identifier: 0x000c3c53

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048   976773119   488385536   8e  Linux LVM

Disk /dev/mapper/centos-root: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/all-home: 3763.8 GB, 3763842580480 bytes, 7351255040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

## 开始扩展

首先将 `/dev/sdc1` 转换为物理卷(PV)

```shell
sudo pvcreate /dev/sdc1 /dev/sde
```

```shell
WARNING: ext4 signature detected on /dev/sdc1 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/sdc1.
  Physical volume "/dev/sdc1" successfully created.
```

查看物理卷(PV)信息

```shell
sudo pvs
```

```shell
  PV         VG     Fmt  Attr PSize    PFree   
  /dev/sda1  all    lvm2 a--  <465.76g       0 
  /dev/sdb1  all    lvm2 a--    <2.73t       0 
  /dev/sdc1         lvm2 ---  <465.76g <465.76g
  /dev/sdd2  all    lvm2 a--  <245.08g       0 
  /dev/sdd3  centos lvm2 a--    52.00g    4.00m
```

将物理卷加入卷组(VG) `all`

```shell
sudo vgextend all /dev/sdc1
```

```shell
Volume group "all" successfully extended
```

查看物理卷(PV)信息

```shell
sudo pvs
```

```shell
  PV         VG     Fmt  Attr PSize    PFree   
  /dev/sda1  all    lvm2 a--  <465.76g       0 
  /dev/sdb1  all    lvm2 a--    <2.73t       0 
  /dev/sdc1  all    lvm2 a--  <465.76g <465.76g
  /dev/sdd2  all    lvm2 a--  <245.08g       0 
  /dev/sdd3  centos lvm2 a--    52.00g    4.00m
```

查看逻辑卷(LV)信息

```shell
sudo lvs
```

```shell
  LV   VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home all    -wi-ao----  3.42t                                                    
  root centos -wi-ao---- 50.00g                                                    
  swap centos -wi-ao----  2.00g  
```

看起来容量并没有变大，这是因为还需要将卷组中的空闲空间扩展到 `/home` 中

```shell
sudo lvextend -L +465.75g /dev/mapper/all-home
```

```shell
  Size of logical volume all/home changed from 3.42 TiB (897370 extents) to <3.88 TiB (1016602 extents).
  Logical volume all/home successfully resized.
```

查看逻辑卷信息(LV)，可以发现空间已经变大了

```shell
sudo lvs
```

```shell
  LV   VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home all    -wi-ao---- <3.88t                                                    
  root centos -wi-ao---- 50.00g                                                    
  swap centos -wi-ao----  2.00g   
```

查看物理卷(PV)信息，可以发现物理卷的可用空间已经变小了

```shell
sudo pvs
```

```shell
  PV         VG     Fmt  Attr PSize    PFree
  /dev/sda1  all    lvm2 a--  <465.76g    0 
  /dev/sdb1  all    lvm2 a--    <2.73t    0 
  /dev/sdc1  all    lvm2 a--  <465.76g 8.00m
  /dev/sdd2  all    lvm2 a--  <245.08g    0 
  /dev/sdd3  centos lvm2 a--    52.00g 4.00m
```

使扩容生效 `xfs_groufs` 针对 xfs文件系统

```shell
mount
sudo xfs_growfs /dev/mapper/all-home
```

查看剩余空间信息

```shell
df -h
```

```shell
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G  2.0G   49G   4% /
devtmpfs                 848M     0  848M   0% /dev
tmpfs                    860M     0  860M   0% /dev/shm
tmpfs                    860M   17M  843M   2% /run
tmpfs                    860M     0  860M   0% /sys/fs/cgroup
/dev/mapper/all-home     3.9T  3.2T  701G  83% /home
/dev/sdd1                976M  136M  774M  15% /boot
tmpfs                    172M     0  172M   0% /run/user/1000
```

`/home` 空间变大了0.4T,扩容成功

## 参考

https://blog.csdn.net/yaofengyaofeng/article/details/82353282

https://blog.csdn.net/u012439646/article/details/73380197

http://lzw.me/a/linux-lvm.html

https://blog.csdn.net/youjin/article/details/79137203

https://blog.csdn.net/qq_27281257/article/details/81603410

https://blog.csdn.net/weixin_42350212/article/details/80570211