#OpenStack Controller Node Installation

##主机名修改
修改主机名为controller，此步骤在network以及computer节点同样需要执行

```
# hostnamectl set-hostname controller
```
无需重启，重新登录即可

##主机域名解析
修改```/etc/hosts```文件，新增节点域名对应（注：所有虚拟机节点均需要进行设置）

```
[root@controller ~]# vim /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.0.0.11   controller
10.0.0.21   network
10.0.0.31   computer1
```

##网卡设置
为确保实践环境中域名解析以及相关服务正常，节点的网卡地址均采用静态网络地址

根据规划的IP地址，controller节点IP：

 IP address: 10.0.0.11

 Network mask: 255.255.255.0 (or /24)

 Default gateway: 10.0.0.2
 
修改```/etc/sysconfig/network-scripts/ifcfg-INTERFACE_NAME```文件，本例中```INTERFACE_NAME```为*eno16777736* 
 
需要修改的配置项如下，其他不变

```
BOOTPROTO=static
IPADDR=10.0.0.11
PREFIX=24
GATEWAY=10.0.0.2
DNS1=10.0.0.2

ONBOOT=yes
```

##NTP 设置
chrony是网络时间协议 NTP 的客户端和服务器软件。该程序可以让你的计算机时间保持最精确。

安装chrony服务

```
# yum install chrony -y
```

编辑```/etc/chrony.conf```文件

```
#替换NTP_SERVER为可用的IP地址或主机名即可，本机使用默认即可
server NTP_SERVER iburst

#添加子网地址，使其他节点可用本机NTP server服务
allow 10.0.0.0/24
```

启动服务，并将其配置为随系统自启动

```
# systemctl enable chronyd.service
# systemctl start chronyd.service
```

检查NTP同步

```
# chronyc sources
210 Number of sources = 3
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? frontier.innolan.net          0   7     0   10y     +0ns[   +0ns] +/-    0ns
^* 120.25.115.19                 2   6    17    53   +283us[+1638us] +/-   48ms
^? 2001:da8:9000::130            0   6     0   10y     +0ns[   +0ns] +/-    0ns
```

##OpenStack源

####添加OpenStack newton官方源

```
# yum install centos-release-openstack-newton -y 
# yum upgrade
```
如果系统更新中包含kernel的更新，需要重启系统

####安装OpenStack client

```
# yum install python-openstackclient -y
```
此OpenStack命令行安装之后可以直接使用CLI```openstack```执行相关命令
####安装OpenStack SELinux

```
# yum install openstack-selinux -y
```
CentOS默认是开启SELinux的，安装此包合并OpenStack服务的安全策略


##YUM源设置
CentOS默认使用的是官方YUM源地址，通常我们在下载其他软件包或者后面OpenStack相关包的时候会比较慢，因此此处将YUM源地址改为[阿里云aliyun](http://mirrors.aliyun.com/)的镜像地址，以加快安装（也可以使用[网易163](http://mirrors.163.com/)的源，速度也非常好）
####备份

```
# mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```
####下载新的CentOS-Base.repo 到/etc/yum.repos.d/

```
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

####修改OpenStack Repo ```/etc/yum.repos.d/CentOS-OpenStack-newton.repo```
[centos-openstack-newton]的baseurl修改为aliyun的地址即可，其他config项可不变

```
[centos-openstack-newton]
name=CentOS-7 - OpenStack newton
#baseurl=http://mirror.centos.org/centos/7/cloud/$basearch/openstack-newton/
baseurl=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-newton/
gpgcheck=1
enabled=1
#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
```

####更新源建立本地缓存

```
# yum clean all
# yum repolist
# yum makecache
```

##Controller节点部署
***
###数据库
####1.安装数据库包

```
# yum install mariadb mariadb-server python2-PyMySQL -y
```
####2.新增并修改数据库配置文件```/etc/my.cnf.d/openstack.cnf```
增加[mysqld]部分，添加配置如下：

```
[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
####3.添加数据库服务开机启动

```
# systemctl enable mariadb.service
# systemctl start mariadb.service
```
####4.更改数据库安全并添加root密码

```
# mysql_secure_installation
```
执行上述脚本之后，交互式设置数据库root密码，此密码在之后操作数据库的时候均会使用到，但不会出现在配置文件中。本实践设置密码为”*openstack*“，具体可见[简介](readme.md)Passwords章节。

***

###消息队列 Message queue
消息队列服务一般运行在控制节点上，用于协调操作和服务间的状态。OpenStack支持多种类型的消息队列，包括RabbitMQ、Qpid、ZeroMQ等等。

本实践采用的是RabbitMQ消息队列服务。
####1.安装消息队列包

```
# yum install rabbitmq-server -y
```
####2.添加开机运行服务

```
# systemctl enable rabbitmq-server.service
# systemctl start rabbitmq-server.service
```
####3.添加openstack用户

```
# rabbitmqctl add_user openstack RABBIT_PASS
```
用自定义的密码替换```RABBIT_PASS```，本例中使用“*rabbit*”作为openstack用户的密码，具体可见[简介](readme.md)Passwords章节。
####4.配置权限

```
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
将openstack用户添加读和写权限

***

###缓存服务
Identity认证服务使用Memcached来缓存临牌，一般部署在controller节点上

####安装Memcached包并设置为服务开机启动

```
yum install memcached python-memcached -y
```
***

###认证服务 Keystone
OpenStack认证服务整合了认证管理、授权管理以及目录服务，为OpenStack的其他服务提供通用的API。当用户有请求时，Identity服务会验证该用户是否有权限进行此次请求。

Identity服务包含三大组件，服务器、驱动以及模块

####1.添加数据库
使用root用户连接数据库服务器，创建keystone数据库

```
# mysql -u root -p

MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> exit
```
使用自定义密码替换```KEYSTONE_DBPASS ```，本实例使用”*keystone*“作为密码，具体可见[简介](readme.md)Passwords章节。

####2.生成令牌
此处生成一个随机值在初始的配置中作为管理员的令牌

```
# openssl rand -hex 10
```

####3.安装配置Identity Keystone组件
#####安装
```
# yum install openstack-keystone httpd mod_wsgi -y 
```
#####编辑```/etc/keystone/keystone.conf```配置文件
```
[DEFAULT]
...
admin_token = ADMIN_TOKEN

[database]
...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
...
provider = fernet
```
注：```ADMIN_TOKEN```字段使用前面步骤生成的随机数替换，```KEYSTONE_DBPASS```使用keystone数据库密码替换，具体可见[简介](readme.md)Passwords章节。
#####初始化并且同步keystone数据库
```
# su -s /bin/sh -c "keystone-manage db_sync" keystone
```
#####初始化fernet key仓库
```
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
#####引导Identity Keystone服务
```
# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
```ADMIN_PASS```使用admin用户密码替换，本例使用”*admin*“，具体可见[简介](readme.md)Passwords章节。
#####加入开机启动
```
# systemctl enable openstack-keystone.service
# systemctl start openstack-keystone.service
```

####4.部署Apache HTTP Server
#####修改```/etc/httpd/conf/httpd.conf```配置文件
```
ServerName controller
```
#####建立软连接
```
# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```
#####加入开机启动
```
# systemctl enable httpd.service
# systemctl start httpd.service
```
####5.创建域、项目、用户和角色
身份认证服务为每个OpenStack服务提供认证服务，使用了domains, projects, users和roles组合
#####创建service工程
本例使用一个service项目，这个项目所包含的每一项服务都有独立的用户

```
# openstack project create --domain default \
  --description "Service Project" service
```
#####创建配置demo项目和用户
常规（非管理）任务应该使用无特权的项目和用户

```
# openstack project create --domain default --description "Demo Project" demo
  
# openstack user create --domain default --password-prompt demo
  
# openstack role create user

# openstack role add --project demo --user demo user
```
####6.验证操作
#####关闭临时认证令牌机制
编辑```/etc/keystone/keystone-paste.ini```文件，将```admin_token_auth```项从```[pipeline:public_api]```，```[pipeline:admin_api]```以及```[pipeline:api_v3]```部门删除。
#####重置环境变量
```
# unset OS_AUTH_URL OS_PASSWORD
```
#####请求令牌认证
*admin用户*

```
# openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue
```
提示输入admin用户的密码，即“ADMIN_PASS”，本例使用”*admin*“，具体可见[简介](readme.md)Passwords章节。

*demo用户*

```
# openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue
```
提示输入demo用户的密码，即“DEMO_PASS”，本例使用”*demo*“，具体可见[简介](readme.md)Passwords章节。

**注**：命令执行令牌认证后均有返回打印，验证成功
####7.创建环境变量脚本
创建*admin*和*demo*用户环境变量脚本，提升客户端操作效率
#####创建并编辑```admin-openrc```文件
```
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
其中```ADMIN_PASS```使用admin用户密码替换，本例使用”*admin*“，具体可见[简介](readme.md)Passwords章节。
#####创建并编辑```demo-openrc```文件
```
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
其中```DEMO_PASS```使用demo用户密码替换，本例使用”*demo*“，具体可见[简介](readme.md)Passwords章节。
#####使用方法
```
# source admin-openrc
或
# . admin-openrc
```
***
###镜像服务 Glance
镜像服务 (glance) 允许用户发现、注册和获取虚拟机镜像。它提供了一个 REST API，允许用户查询虚拟机镜像的 metadata 并获取一个真实的镜像。用户可以将虚拟机镜像存储到各种位置，从简单的文件系统到对象存储系统并通过镜像服务使用。

管理节点上默认使用的镜像存储路径为```/var/lib/glance/images/```

OpenStack的镜像服务是IAAS的核心组件，它接受磁盘镜像或服务器镜像API请求，和来自终端用户或OpenStack计算组件的元数据定义。同时，它也支持包括OpenStack对象存储在内的多种类型仓库上的磁盘镜像或服务器镜像存储。

OpenStack镜像服务包括以下组件，glance-api、glance-registry、数据库、镜像文件的存储仓库以及元数据定义服务。

####1.添加glance数据库
使用root用户连接数据库服务器，创建glance数据库

```
# mysql -u root -p

MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> exit
```
使用自定义密码替换```GLANCE_DBPASS```，本实例使用”*glance*“作为密码，具体可见[简介](readme.md)Passwords章节。
####2.创建服务证书
此步骤需获取admin管理员凭证，才能有执行部分命令的权限

创建glance用户之后，需要将admin角色添加到glance用户以及service项目上，然后创建glance服务实体。

```
# source admin-openrc

# openstack user create --domain default --password-prompt glance

# openstack role add --project service --user glance admin

# openstack service create --name glance --description "OpenStack Image" image
```
####3.创建镜像服务的API端点
```
# openstack endpoint create --region RegionOne \
  image public http://controller:9292
  
# openstack endpoint create --region RegionOne \
  image internal http://controller:9292

# openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```
####4.安装配置glance组件
#####安装软件包
```
# yum install openstack-glance -y
```
#####编辑```/etc/glance/glance-api.conf```文件
```
[database]
...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone

[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
注：```GLANCE_DBPASS```字段使用glance镜像服务中数据库的密码替换，```GLANCE_PASS```使用认证服务中glance用户的密码替换，具体可见[简介](readme.md)Passwords章节。
#####编辑```/etc/glance/glance-registry.conf```文件
```
[database]
...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone
```
注：```GLANCE_DBPASS```字段使用glance镜像服务中数据库的密码替换，```GLANCE_PASS```使用认证服务中glance用户的密码替换，具体可见[简介](readme.md)Passwords章节。
#####写入镜像服务数据库
```
# su -s /bin/sh -c "glance-manage db_sync" glance
```
忽略输出的告警
####5.配置开机启动
```
# systemctl enable openstack-glance-api.service openstack-glance-registry.service
# systemctl start openstack-glance-api.service openstack-glance-registry.service
```
####6.验证操作
使用官方的[CirrOS](https://launchpad.net/cirros)对镜像服务进行验证

获取admin凭证来获取只有管理员能执行的命令的访问权限

```
# source admin-openrc
```
下载镜像，如果下载速度较慢，可以在宿主机上下载完成后上传至虚拟机当中

```
# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```
使用qcow2磁盘格式，bare容器格式上传到镜像服务器并设置为公共可见

```
# openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare --public
```
确认镜像的上传并验证属性

```
# openstack image list
```
有镜像结果输出即验证成功
***

###计算服务 Compute
OpenStack计算服务用来托管和管理云计算系统，是IAAS系统的主要组成部分。计算组件请求OpenStack Identity服务进行认证；请求OpenStack Image服务提供磁盘镜像；为OpenStack dashboard提供用户与管理员接口。

OpenStack计算服务包含以下组件：nova-api、nova-api-metadata、nova-compute、nova-scheduler等服务，nova-conductor、nova-cert等模块，nova-network worker、nova-consoleauth、nova-novncproxy、nova-spicehtml5proxy、nova-xvpvncproxy、nova-cert等守护进程，nova客户端，队列以及数据库等。
####1.添加nova以及nova_api数据库

```
# mysql -u root -p

MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
```
使用自定义密码替换```NOVA_DBPASS```，本实例使用”*nova*“作为密码，具体可见[简介](readme.md)Passwords章节。
####2.创建服务证书
获取admin凭证后，创建nova用户并给nova用户添加admin角色，最后创建服务实体

```
# source admin-openrc

# openstack user create --domain default --password-prompt nova

# openstack role add --project service --user nova admin

# openstack service create --name nova --description "OpenStack Compute" compute
```
####3.创建计算compute服务的API端点endpoints

```
# openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1/%\(tenant_id\)s
  
# openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1/%\(tenant_id\)s
  
# openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1/%\(tenant_id\)s
```
####4.安装并配置计算Compute组件
#####安装软件包

```
# yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler -y
```
#####编辑```/etc/nova/nova.conf```配置文件

```
[DEFAULT]
...
auth_strategy = keystone
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 10.0.0.11
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
...
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
...
api_servers = http://controller:9292

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp
```
注：```RABBIT_PASS```字段使用在 “RabbitMQ” 中为 “openstack” 选择的密码替换，```NOVA_DBPASS```使用为 Compute 数据库选择的密码替换，```NOVA_PASS```使用在身份认证服务中设置的”nova“用户的密码替换，具体可见[简介](readme.md)Passwords章节。


#####写入计算Compute服务数据库

```
# su -s /bin/sh -c "nova-manage api_db sync" nova
# su -s /bin/sh -c "nova-manage db sync" nova
```
####5.完成安装设置开机启动

```
# systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
# systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```






