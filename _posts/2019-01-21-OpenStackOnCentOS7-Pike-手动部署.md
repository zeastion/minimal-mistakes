---
title: "OpenStack Pike On CentOS7 手动部署"
categories:
  - 云计算
date: 2019-1-21
toc: true
toc_label: "部署过程"
toc_icon: "align-left"
header:
  teaser: /assets/images/fbs_teaser.gif
---

> OpenStack 版本 Pike & CentOS 版本 7.5.1804

## 环境

### 网络

- 控制节点

  controller01

  Interface 1： 10.0.0.11 ( Management network )

  Interface 2： 192.168.52.142 ( Provider network )

- 计算节点

  computer01

  Interface 1： 10.0.0.31

  Interface 2： 192.168.52.143

- 块存储节点

  block01

  Interface 1： 10.0.0.41

- 对象存储节点

  object01

  Interface 1： 10.0.0.51

  object02

  Interface 1： 10.0.0.52

### 域名解析

- 各节点配置 hosts 文件

  ```
  [root@allnodes ~]# more /etc/hosts

  #Controller
  10.0.0.11       controller01

  #Computer
  10.0.0.31       computer01

  #Block
  10.0.0.41       block01

  #Object
  10.0.0.51       object01
  10.0.0.52       object02
  ```

### 时间同步

- controller01

  安装

  ```
  # yum install chrony -y

  # vim /etc/chrony.conf
  ...
  allow 10.0.0.0/24
  ...

  # systemctl enable chronyd.service

  # systemctl start chronyd.service
  ```

  验证

  ```
  # chronyc sources
  210 Number of sources = 4
  MS Name/IP address         Stratum Poll Reach LastRx Last sample
  ===============================================================================
  ^+ 85.199.214.101                1  10   377   195  +3860us[+6391us] +/-  117ms
  ^+ 85.199.214.100                1  10   377   179    -10ms[-8917us] +/-  122ms
  ^* 119.28.183.184                2   6   277    57  +3327us[+3863us] +/-   76ms
  ^+ uk.cluster.ntp.faelix.net     2  10   137   612  -1256us[+9007us] +/-  147ms
  ```

- 其它节点

  安装

  ```
  # yum install chrony -y

  # vim /etc/chrony.conf
  ...
  server controller01 iburst
  ...

  # systemctl enable chronyd.service

  # systemctl start chronyd.service
  ```

  验证

  ```
  # chronyc sources
  210 Number of sources = 1
  MS Name/IP address         Stratum Poll Reach LastRx Last sample
  ===============================================================================
  ^? controller01                  0  10     0     -     +0ns[   +0ns] +/-    0ns
  ```

### OpenStack 包

- 在所有节点执行

  ```
  # yum install centos-release-openstack-pike

  # yum upgrade

  # reboot

  # yum install python-openstackclient

  # yum install openstack-selinux
  ```

### 数据库

- controller01

  ```
  # yum install mariadb mariadb-server python2-PyMySQL -y

  # vim /etc/my.cnf.d/openstack.cnf
  [mysqld]
  bind-address = 10.0.0.11

  default-storage-engine = innodb
  innodb_file_per_table = on
  max_connections = 4096
  collation-server = utf8_general_ci
  character-set-server = utf8

  # systemctl enable mariadb.service

  # systemctl start mariadb.service

  # mysql_secure_installation
  ```

### 消息队列 Rabbitmq

- controller01

  ```
  # yum install rabbitmq-server -y

  # systemctl enable rabbitmq-server.service

  # systemctl start rabbitmq-server.service

  # rabbitmqctl add_user openstack qwe123
  Creating user "openstack" ...

  # rabbitmqctl set_permissions openstack ".*" ".*" ".*"
  Setting permissions for user "openstack" in vhost "/" ...
  ```

### 缓存服务 Memcached

- controller01

  ```
  # yum install memcached python-memcached -y

  # vim /etc/sysconfig/memcached
  PORT="11211"
  USER="memcached"
  MAXCONN="1024"
  CACHESIZE="64"
  OPTIONS="-l 127.0.0.1,::1,controller01"

  # systemctl enable memcached.service

  # systemctl start memcached.service
  ```

### 分布式键值存储 Etcd

- controller01

  ```
  # yum install etcd -y

  # vim /etc/etcd/etcd.conf
  #[Member]
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="http://10.0.0.11:2380"
  ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
  ETCD_NAME="controller"
  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
  ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
  ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
  ETCD_INITIAL_CLUSTER_STATE="new"

  # systemctl enable etcd

  # systemctl start etcd
  ```


## Keystone

### 安装与配置

出于可伸缩性目的，部署 Fernet 令牌和 Apache HTTP 服务器处理请求

1. 准备

   ```
   # mysql -uroot -p

   MariaDB [(none)]> CREATE DATABASE keystone;
   Query OK, 1 row affected (0.00 sec)

   MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'qwe123';
   Query OK, 0 rows affected (0.00 sec)

   MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'qwe123';
   Query OK, 0 rows affected (0.00 sec)

   MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller01' IDENTIFIED BY 'qwe123';
   Query OK, 0 rows affected (0.00 sec)

   MariaDB [(none)]> quit
   Bye
   ```

2. 组件

   ```
   # yum install openstack-keystone httpd mod_wsgi -y

   # vim /etc/keystone/keystone.conf
   ...
   [database]
   connection = mysql+pymysql://keystone:qwe123@controller01/keystone
   ...
   [token]
   provider = fernet
   ```

   填充身份认证服务数据库

   ```
   # sh -c "keystone-manage db_sync" keystone
   ```

   初始化 Fernet 密钥存储库

   ```
   # keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

   # keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
   ```

   引导身份服务
   ```
   # keystone-manage bootstrap --bootstrap-password qwe123 \
   --bootstrap-admin-url http://controller01:35357/v3/ \
   --bootstrap-internal-url http://controller01:5000/v3/ \
   --bootstrap-public-url http://controller01:5000/v3/ \
   --bootstrap-region-id RegionOne
   ```

   Apache HTTP

   ```
   # vim /etc/httpd/conf/httpd.conf
   ...
   ServerName controller01
   ···

   # ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

   # systemctl enable httpd.service

   # systemctl start httpd.service
   ```

   变量文件

   ```
   # vim admin_openrc
   export OS_USERNAME=admin
   export OS_PASSWORD=qwe123
   export OS_PROJECT_NAME=admin
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_AUTH_URL=http://controller01:35357/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2

   # source admin_openrc
   ```

### 认证信息

1. 创建 'service' 项目

   ```
   # openstack project create --domain default --description "Service Project" service
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | Service Project                  |
   | domain_id   | default                          |
   | enabled     | True                             |
   | id          | 5f81944c65c346efa9ff8e0753ba795f |
   | is_domain   | False                            |
   | name        | service                          |
   | parent_id   | default                          |
   +-------------+----------------------------------+
   ```

2. 建立无特权的常规项目和用户 (demo)

   创建 'demo' 项目

   ```
   # openstack project create --domain default --description "Demo Project" demo
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | Demo Project                     |
   | domain_id   | default                          |
   | enabled     | True                             |
   | id          | 40290ff56e254126bcc41f58536fd20e |
   | is_domain   | False                            |
   | name        | demo                             |
   | parent_id   | default                          |
   +-------------+----------------------------------+
   ```

   创建 'demo' 用户

   ```
   # openstack user create --domain default --password-prompt demo
   User Password:
   Repeat User Password:
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | afe36381927c4e58a2128cce7ef488e8 |
   | name                | demo                             |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+
   ```

   创建 'user' 角色

   ```
   # openstack role create user
   +-----------+----------------------------------+
   | Field     | Value                            |
   +-----------+----------------------------------+
   | domain_id | None                             |
   | id        | e19d103864f04adc80e25f41b3610f86 |
   | name      | user                             |
   +-----------+----------------------------------+
   ```

   给项目和用户赋予 'user' 角色

   ```
   # openstack role add --project demo --user demo user
   ```

3. 验证


   获取 'admin' 用户认证令牌

   ```
   # unset OS_AUTH_URL OS_PASSWORD

   # openstack --os-auth-url http://controller01:35357/v3 \
   --os-project-domain-name Default \
   --os-user-domain-name Default \
   --os-project-name admin \
   --os-username admin token issue
   Password:
   +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   | Field      | Value                                                                                                                                                                                   |
   +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   | expires    | 2019-01-22T08:32:09+0000                                                                                                                                                                |
   | id         | gAAAAABcRsb52XP415xIEw6b_MzmXzcJofMyFonivsNlBRHGWc2OSlWKJCUgUFou_1SSXTqJdnsxCjOznWgSgh7GbsqsKYRCi6hrUVTWvAM0KozaLqq_PTy_u7iSRBAoQ8QAgPg6yCOcdJJ7IniI_fioULe3pH-WVxqWOlW7urwc7X1_AjVA9tE |
   | project_id | 0a7aef2ccc8b4a71b80534b7c43fe8ab                                                                                                                                                        |
   | user_id    | 3197614029de4d48ab1d4ae76a9776e4                                                                                                                                                        |
   +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   ```

   获取 'demo' 用户认证令牌

   ```
   # openstack --os-auth-url http://controller01:5000/v3 \
   --os-project-domain-name Default \
   --os-user-domain-name Default \
   --os-project-name demo \
   --os-username demo token issue
   Password:
   +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   | Field      | Value                                                                                                                                                                                   |
   +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   | expires    | 2019-01-22T08:37:12+0000                                                                                                                                                                |
   | id         | gAAAAABcRsgoyY1hy_ZnA_OpNP7nyywOzm3qs7AcnxqSCwNBTqzBPvy0BUIga5z1teJUgMC8PRyuzQdg3O5lKUJgbJ8NS5mge2_ZGVaS4dKFiZabRTU2Oncja15MIMIUNeyTpwxUf33waafnzvsK80Ues6_vhxj9nkQboAYTqN57MLGde6eivLU |
   | project_id | 40290ff56e254126bcc41f58536fd20e                                                                                                                                                        |
   | user_id    | afe36381927c4e58a2128cce7ef488e8                                                                                                                                                        |
   +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   ```

4. 环境脚本

   admin 脚本

   ```
   # more admin_openrc
   export OS_USERNAME=admin
   export OS_PASSWORD=qwe123
   export OS_PROJECT_NAME=admin
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_AUTH_URL=http://controller01:35357/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   ```

   demo 脚本

   ```
   # vim demo_openrc
   export OS_PROJECT_DOMAIN_NAME=default
   export OS_USER_DOMAIN_NAME=default
   export OS_PROJECT_NAME=demo
   export OS_USERNAME=demo
   export OS_PASSWORD=qwe123
   export OS_AUTH_URL=http://controller01:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   ```

   验证

   ```
   # . admin_openrc

   # openstack token issue
   ```

## Glance

### 安装与配置

1. 准备

   创建数据库

   ```
   # mysql -uroot -p

   MariaDB [(none)]> CREATE DATABASE glance;
   Query OK, 1 row affected (0.00 sec)

   MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'qwe123';
   Query OK, 0 rows affected (0.00 sec)

   MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'qwe123';
   Query OK, 0 rows affected (0.00 sec)

   MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller01' IDENTIFIED BY 'qwe123';
   Query OK, 0 rows affected (0.00 sec)

   MariaDB [(none)]> quit
   Bye
   ```

   凭证

   ```
   # . admin_openrc

   # openstack user create --domain default --password-prompt glance
   User Password:
   Repeat User Password:
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | f0462a999e68425fb4c9f34100088932 |
   | name                | glance                           |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+

   # openstack role add --project service --user glance admin

   # openstack service create --name glance --description "OpenStack Image" image
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | OpenStack Image                  |
   | enabled     | True                             |
   | id          | 92256da4a9e846adba12de8eead309c4 |
   | name        | glance                           |
   | type        | image                            |
   +-------------+----------------------------------+
   ```

   创建 Image service 的 API endpoints

   ```
   # openstack endpoint create --region RegionOne image public http://controller01:9292
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | 90d2fc3020d34ff0b7c0679325a44860 |
   | interface    | public                           |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | 92256da4a9e846adba12de8eead309c4 |
   | service_name | glance                           |
   | service_type | image                            |
   | url          | http://controller01:9292         |
   +--------------+----------------------------------+

   # openstack endpoint create --region RegionOne image internal http://controller01:9292
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | 0c2a2da407e74b43b911244d773b843d |
   | interface    | internal                         |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | 92256da4a9e846adba12de8eead309c4 |
   | service_name | glance                           |
   | service_type | image                            |
   | url          | http://controller01:9292         |
   +--------------+----------------------------------+

   # openstack endpoint create --region RegionOne image admin http://controller01:9292
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | a4bfc557d5004996945316093edde41f |
   | interface    | admin                            |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | 92256da4a9e846adba12de8eead309c4 |
   | service_name | glance                           |
   | service_type | image                            |
   | url          | http://controller01:9292         |
   +--------------+----------------------------------+
   ```

2. 组件

   ```
   # yum install openstack-glance -y

   # vim /etc/glance/glance-api.conf
   ...
   [database]
   connection = mysql+pymysql://glance:qwe123@controller01/glance
   ...
   [glance_store]
   ...
   stores = file,http
   default_store = file
   filesystem_store_datadir = /var/lib/glance/images/
   ...
   [keystone_authtoken]
   auth_uri = http://controller01:5000
   auth_url = http://controller01:35357
   memcached_servers = controller01:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = glance
   password = qwe123
   ...
   [paste_deploy]
   flavor = keystone
   ...

   # vim /etc/glance/glance-registry.conf
   ...
   [database]
   ...
   connection = mysql+pymysql://glance:qwe123@controller01/glance
   ...
   [keystone_authtoken]
   ...
   auth_uri = http://controller01:5000
   auth_url = http://controller01:35357
   memcached_servers = controller01:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = glance
   password = qwe123
   ...

   [paste_deploy]
   ...
   flavor = keystone
   ```

   填充 Image 服务数据库

   ```
   # su -s /bin/sh -c "glance-manage db_sync" glance
   /usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:1328: OsloDBDeprecationWarning: EngineFacade is deprecated; please use oslo_db.sqlalchemy.enginefacade
   expire_on_commit=expire_on_commit, _conf=conf)
   INFO  [alembic.runtime.migration] Context impl MySQLImpl.
   INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
   INFO  [alembic.runtime.migration] Running upgrade  -> liberty, liberty initial
   INFO  [alembic.runtime.migration] Running upgrade liberty -> mitaka01, add index on created_at and updated_at columns of 'images' table
   INFO  [alembic.runtime.migration] Running upgrade mitaka01 -> mitaka02, update metadef os_nova_server
   INFO  [alembic.runtime.migration] Running upgrade mitaka02 -> ocata01, add visibility to and remove is_public from images
   INFO  [alembic.runtime.migration] Running upgrade ocata01 -> pike01, drop glare artifacts tables
   INFO  [alembic.runtime.migration] Context impl MySQLImpl.
   INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
   Upgraded database to: pike01, current revision(s): pike01
   ```

   启动服务

   ```
   # systemctl enable openstack-glance-api.service openstack-glance-registry.service

   # systemctl start openstack-glance-api.service openstack-glance-registry.service
   ```
