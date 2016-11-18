# 云徙标准虚拟机镜像制作

## 目录
* **centos**
	* [centos7.2](#centos-7.2)
* **ubuntu**
	* [ubuntu xx](#ubuntu-xx)
* **windows**
	* [windows server 2008](#win-server-2008)



## <span id="centos-7.2">centos7.2</span>

###1. 系统安装

####a. 时区
	
选择中国地区 CST 或者 安装完成后手动修改
	
```	
[root@172-16-20-169 ~]# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
[root@172-16-20-169 ~]# date
2016年 11月 18日 星期五 10:21:40 CST	
[root@172-16-20-169 ~]# date -R
Fri, 18 Nov 2016 10:21:43 +0800
```

####b. 安装类型
	
选择```computer node```，全选额外安装包即可，将常用的一些debug或者工具顺带安装
	
####c. 网络选择

将首块网卡```eth0```设置为开机自启动

####d. 其他

设置root密码，默认设置为```root```

	
###2. 常用设置变更

####a. ssh
修改```/etc/ssh/sshd_config``` 配置文件，更改以下内容

```
PermitRootLogin yes
GSSAPIAuthentication no
UseDNS no
```

####b. network
修改```/etc/sysconfig/network-scripts/ifcfg-eth0```文件，更改以下内容

```
BOOTPROTO="dhcp"
ONBOOT="yes"
PEERDNS="no"
IPV6_PEERDNS="no"
```
修改```/etc/resolv.conf```配置文件，删除```nameserver xxx.xxx.xxx.xxx```相关内容

ipforward 开关打开

```
[root@172-16-20-169 ~]# cat /proc/sys/net/ipv4/ip_forward
0
[root@172-16-20-169 ~]# echo 1 > /proc/sys/net/ipv4/ip_forward
[root@172-16-20-169 ~]# cat /proc/sys/net/ipv4/ip_forward
1
```

####c. security
关闭防火墙相关服务

```
systemctl stop firewalld.service
systemctl disable firewalld.service
```
清除iptables规则

```
iptables -F
```

关闭Selinux

```
# 临时关闭
setenforce 0

# 修改文件关闭
/etc/selinux/config中将SELINUX=enforcing改为SELINUX=disabled
重启虚拟机
```

## <span id="ubuntu-xx">ubuntu xx</span>
to do

## <span id="win-server-2008">windows server 2008</span>
to do
