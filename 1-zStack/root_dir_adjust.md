#CentOS调整ROOT目录大小的方法

##背景
CentOS系统在某些默认安装情况下，目录的空间划分是固定的，一般给root只有50G左右，而home目录却非常大。而在许多实际场景中，用户应用程序会部署在根目录，日志文件也会不断扩大，导致root目录被占用至满的情况时常发生。因此本文简单介绍如何调整root根目录的方法。

**注**：本文介绍的方法是通过减少/home目录的方法来调整根目录，且虚拟机磁盘的格式为qcow2

##调整方法
###1.查看当前分区的划分情况
```
# df -Th

/dev/mapper/centos-root xfs        50G   50G  100M   100% /
devtmpfs                devtmpfs  7.8G     0  7.8G    0% /dev
tmpfs                   tmpfs     7.8G     0  7.8G    0% /dev/shm
tmpfs                   tmpfs     7.8G   25M  7.8G    1% /run
tmpfs                   tmpfs     7.8G     0  7.8G    0% /sys/fs/cgroup
/dev/mapper/centos-home xfs        92G   33M   92G    1% /home
/dev/vda1               xfs       497M  130M  368M   27% /boot
```
可以看到此时根目录已经占满，而/home目录基本没有被使用
###1.切换到root用户，终止占用/home目录的进程
```
# su root
# fuser -m -u -v -k /home
```
###2.备份/home并卸载挂载点
```
# cp -r  /home/  homebak/
# umount /home
```
如果卸载失败，请检查/home目录的进程占用情况
###3.删除逻辑卷
```
# lvdisplay
# lvremove /dev/mapper/centos-home
```
此时就多出了空间可以扩展/root分区目录
###4.扩展/root分区
```
# lvextend -L +80G /dev/mapper/centos-root
```
"+80G"表示增加容量，如果不带“+”号则表示扩展至80G
###5.扩展目录
```
# xfs_growfs /dev/mapper/centos-root
```
###6.恢复/home分区
```
# lvcreate -L 10G -n home centos
# mkfs.xfs /dev/centos/home
# mount /dev/centos/home /home
```
此步骤非常重要，因为原操作系统包含此逻辑卷，并且开机也会挂载对应的目录，如果不执行此步骤，重启系统会失败，CentOS系统镜像可能遭到破坏。
***
至此，通过减少/home目录来扩展/root目录的步骤完成，查询分区状况可见

```
# df -h

/dev/mapper/centos-root  132G   50G   83G   38% /
devtmpfs                 7.8G     0  7.8G    0% /dev
tmpfs                    7.8G     0  7.8G    0% /dev/shm
tmpfs                    7.8G   25M  7.8G    1% /run
tmpfs                    7.8G     0  7.8G    0% /sys/fs/cgroup
/dev/mapper/centos-home  9.6G   33M  9.6G    1% /home
/dev/vda1                497M  130M  368M   27% /boot
```