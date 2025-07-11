+++
date = '2025-07-11T09:29:37+08:00'
draft = false
title = 'LVM调整调整分区'
+++

Linux 系统默认分区大小通常面临/home空间太大，/root 空间太小的问题。采用LVM分区管理可以比较方便的调整各部分分区大小。
以下针对默认的ext4分区格式来说明将/home的空间减小一部分扩展到root分区上。
首先，重启系统进入tty2，不开启桌面会话。并以root账户登录，若以普通账户登录相关进程会占用/home分区，导致无法卸载home分区。
使用df -h 查看系统的逻辑卷分区和对应的分区大小，挂载位置，可用空间等情况。
随后，卸载/home分区```
umount /home
```
使用e2fscheck 检查文件系统是否有错误。(必须)
```
e2fscheck -f /dev/ArchinstallVg/home
```
减少/home 分区逻辑卷物理空间大小，不可小于当前已占用空间
```
resize2fs /dev/ArchinstallVg/home 200G #缩减到200G
```
使用lvreduce 或lvresize 缩减逻辑卷大小
```
lvreduce -L [new_size] /dev/ArchinstallVg/home
```

再次检查文件系统是否有误
```
e2fscheck -f /dev/ArchinstallVg/home
```
使用lvextend 将未使用空间扩展到/root分区
```
lvextend -L +%100FREE /dev/ArchinstallVg/root
```
检查文件系统
```
e2fscheck -f /dev/ArchinstallVg/home
```
确认无误后，重新挂载/home分区
```
mount /dev/ArchiinstallVg/root
```
至此，容量调整完毕


