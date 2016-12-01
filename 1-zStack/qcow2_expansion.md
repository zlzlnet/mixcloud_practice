#qcow2 磁盘在线扩容方法

##直接扩展现有qcow2格式磁盘大小的方法

**注:** 对应虚拟机的分区为vda，虚拟机系统为centos7

###1. 查看磁盘文件信息，进行扩展

```
[root@vp-01 export]# qemu-img info test.qcow2
image: test.qcow2
file format: qcow2
virtual size: 20G (21474836480 bytes)
disk size: 2.1G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

新增磁盘容量大小20G

```
[root@vp-01 export]# qemu-img resize test.qcow2 +20G
Image resized.
```

```
[root@vp-01 export]# qemu-img info test.qcow2
image: test.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 2.1G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```
对比后发现磁盘已经由原来的20G变为40G了

###2. 启动虚拟机查看磁盘信息

```
[root@172-16-20-171 ~]#
[root@172-16-20-171 ~]# df -Th
文件系统                类型      容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root xfs        18G  1.9G   16G   11% /
devtmpfs                devtmpfs  487M     0  487M    0% /dev
tmpfs                   tmpfs     497M     0  497M    0% /dev/shm
tmpfs                   tmpfs     497M  6.6M  490M    2% /run
tmpfs                   tmpfs     497M     0  497M    0% /sys/fs/cgroup
/dev/vda1               xfs       497M  130M  368M   27% /boot
tmpfs                   tmpfs     100M     0  100M    0% /run/user/0
[root@172-16-20-171 ~]#
```

###3. 开始分区

```
[root@172-16-20-171 ~]#
[root@172-16-20-171 ~]# fdisk /dev/vda
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。


命令(输入 m 获取帮助)：p

磁盘 /dev/vda：42.9 GB, 42949672960 字节，83886080 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x0003f814

   设备 Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048     1026047      512000   83  Linux
/dev/vda2         1026048    41943039    20458496   8e  Linux LVM

命令(输入 m 获取帮助)：n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p
分区号 (3,4，默认 3)：3
起始 扇区 (41943040-83886079，默认为 41943040)：
将使用默认值 41943040
Last 扇区, +扇区 or +size{K,M,G} (41943040-83886079，默认为 83886079)：
将使用默认值 83886079
分区 3 已设置为 Linux 类型，大小设为 20 GiB

命令(输入 m 获取帮助)：t
分区号 (1-3，默认 3)：3
Hex 代码(输入 L 列出所有代码)：8e
已将分区“Linux”的类型更改为“Linux LVM”

命令(输入 m 获取帮助)：w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: 设备或资源忙.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
正在同步磁盘。
[root@172-16-20-171 ~]#
```
此步骤当中，最终要的地方就是 分配新的区需要修改分区的system id，将分区类型改为LVM（**Hex代码为8e**）。

###4. 创建物理卷、加入卷组、扩展逻辑卷
创建物理卷

```
[root@172-16-20-171 ~]# pvcreate /dev/vda3
  Physical volume "/dev/vda3" successfully created
[root@172-16-20-171 ~]# pvs
  PV         VG     Fmt  Attr PSize  PFree
  /dev/vda2  centos lvm2 a--  19.51g 40.00m
  /dev/vda3         lvm2 ---  20.00g 20.00g
```

加入卷组

```
[root@172-16-20-171 ~]# vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  centos   1   2   0 wz--n- 19.51g 40.00m
[root@172-16-20-171 ~]# vgextend centos /dev/vda3
  Volume group "centos" successfully extended
[root@172-16-20-171 ~]# vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  centos   2   2   0 wz--n- 39.50g 20.04g
```

扩展逻辑卷

```
[root@172-16-20-171 ~]# lvextend -l +100%FREE /dev/centos/root
  Size of logical volume centos/root changed from 17.47 GiB (4472 extents) to 37.50 GiB (9601 extents).
  Logical volume root successfully resized.
[root@172-16-20-171 ~]# resize2fs /dev/centos/root
resize2fs 1.42.9 (28-Dec-2013)
resize2fs: Bad magic number in super-block 当尝试打开 /dev/centos/root 时
找不到有效的文件系统超级块.
```
发现报错，分析原因是因为使用dh -T查看系统分区类型为xfs

因此直接使用xfs_growfs扩展即可

```
[root@172-16-20-171 ~]# xfs_growfs /dev/centos/root
meta-data=/dev/mapper/centos-root isize=256    agcount=4, agsize=1144832 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=4579328, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 4579328 to 9831424
```

###5. 确认磁盘是否增长

```
[root@172-16-20-171 ~]# df -Th
文件系统                类型      容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root xfs        38G  1.9G   36G    6% /
devtmpfs                devtmpfs  487M     0  487M    0% /dev
tmpfs                   tmpfs     497M     0  497M    0% /dev/shm
tmpfs                   tmpfs     497M  6.6M  490M    2% /run
tmpfs                   tmpfs     497M     0  497M    0% /sys/fs/cgroup
/dev/vda1               xfs       497M  130M  368M   27% /boot
tmpfs                   tmpfs     100M     0  100M    0% /run/user/0
```
发现系统容量已经由20G扩展为40G



