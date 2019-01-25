---
title: "OpenStack Pike On CentOS7 手动部署（上）"
categories:
  - 云计算
date: 2019-1-21
toc: true
toc_label: "部署过程"
toc_icon: "align-left"
header:
  teaser: /assets/images/openstacklogo.jpeg
---

> 基础环境 & Keystone & Glance & Nova

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

### OpenStack 仓库

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

### 消息队列 RabbitMQ

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

1. 数据库

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

2. glance 用户相关设置

   创建 glance 用户

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
   ```

   将 glance 用户加入 service 项目，赋予 admin 角色

   ```
   # openstack role add --project service --user glance admin
   ```

   创建 glance 服务实体

   ```
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

3. glance 组件安装配置

   软件包

   ```
   # yum install openstack-glance -y
   ```

   配置文件

   ```
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

4. 启动服务

   ```
   # systemctl enable openstack-glance-api.service openstack-glance-registry.service

   # systemctl start openstack-glance-api.service openstack-glance-registry.service
   ```

### 验证

1. 下载镜像

   ```
   # wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
   ```

2. 上传

   ```
   # openstack image create "cirros-0.3.5" \
   --file cirros-0.3.5-x86_64-disk.img \
   --disk-format qcow2 --container-format bare --public
   +------------------+------------------------------------------------------+
   | Field            | Value                                                |
   +------------------+------------------------------------------------------+
   | checksum         | f8ab98ff5e73ebab884d80c9dc9c7290                     |
   | container_format | bare                                                 |
   | created_at       | 2019-01-22T10:32:20Z                                 |
   | disk_format      | qcow2                                                |
   | file             | /v2/images/3a295fc0-6899-4893-8390-e2217eab3542/file |
   | id               | 3a295fc0-6899-4893-8390-e2217eab3542                 |
   | min_disk         | 0                                                    |
   | min_ram          | 0                                                    |
   | name             | cirros-0.3.5                                         |
   | owner            | 0a7aef2ccc8b4a71b80534b7c43fe8ab                     |
   | protected        | False                                                |
   | schema           | /v2/schemas/image                                    |
   | size             | 13267968                                             |
   | status           | active                                               |
   | tags             |                                                      |
   | updated_at       | 2019-01-22T10:32:20Z                                 |
   | virtual_size     | None                                                 |
   | visibility       | public                                               |
   +------------------+------------------------------------------------------+
   ```

3. 查看

   ```
   # openstack image list
   +--------------------------------------+--------------+--------+
   | ID                                   | Name         | Status |
   +--------------------------------------+--------------+--------+
   | 3a295fc0-6899-4893-8390-e2217eab3542 | cirros-0.3.5 | active |
   +--------------------------------------+--------------+--------+
   ```


## Nova

### controller01

- 安装与配置

  1. 数据库

     ```
     # mysql -uroot -p

     MariaDB [(none)]> CREATE DATABASE nova_api;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> CREATE DATABASE nova;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> CREATE DATABASE nova_cell0;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> quit
     Bye
     ```

  2. nova 用户相关设置

     ```
     # . admin_openrc

     # openstack user create --domain default --password-prompt nova
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | 782c0f18f87f4c389609cdfdb46bda2f |
     | name                | nova                             |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+

     # openstack role add --project service --user nova admin

     # openstack service create --name nova --description "OpenStack Compute" compute
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | OpenStack Compute                |
     | enabled     | True                             |
     | id          | 58b3eeb2ee3946b3a732d3574206818c |
     | name        | nova                             |
     | type        | compute                          |
     +-------------+----------------------------------+
     ```

     创建 Compute service 的 API endpoints

     ```
     # openstack endpoint create --region RegionOne compute public http://controller01:8774/v2.1
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 4df447fcc93e44cab5869cc92e81ba47 |
     | interface    | public                           |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 58b3eeb2ee3946b3a732d3574206818c |
     | service_name | nova                             |
     | service_type | compute                          |
     | url          | http://controller01:8774/v2.1    |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne compute internal http://controller01:8774/v2.1
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 0d6da04cf6684bc6a84fa4b02be5948c |
     | interface    | internal                         |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 58b3eeb2ee3946b3a732d3574206818c |
     | service_name | nova                             |
     | service_type | compute                          |
     | url          | http://controller01:8774/v2.1    |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne compute admin http://controller01:8774/v2.1
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | cae415c82af64ee78df0f1908025ae6f |
     | interface    | admin                            |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 58b3eeb2ee3946b3a732d3574206818c |
     | service_name | nova                             |
     | service_type | compute                          |
     | url          | http://controller01:8774/v2.1    |
     +--------------+----------------------------------+
     ```

  3. placement 用户相关设置

     创建 placement 用户

     ```
     # openstack user create --domain default --password-prompt placement
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | 1e31b3a978974f768321aefc7681372b |
     | name                | placement                        |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+
     ```

     将 placement 用户加入 service 项目，赋予 admin 角色

     ```
     # openstack role add --project service --user placement admin
     ```

     创建 placement 服务实体

     ```
     # openstack service create --name placement --description "Placement API" placement
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | Placement API                    |
     | enabled     | True                             |
     | id          | 5ce8d1adb0d9468d8cbd6585426a397d |
     | name        | placement                        |
     | type        | placement                        |
     +-------------+----------------------------------+
     ```

     创建 placement service 的 API endpoints

     ```
     # openstack endpoint create --region RegionOne placement public http://controller01:8778
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | a0f905bf92fc4c3a80d318210a7a27b5 |
     | interface    | public                           |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 5ce8d1adb0d9468d8cbd6585426a397d |
     | service_name | placement                        |
     | service_type | placement                        |
     | url          | http://controller01:8778         |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne placement internal http://controller01:8778
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | e71eadc8ed8f4fceab52e6d617866163 |
     | interface    | internal                         |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 5ce8d1adb0d9468d8cbd6585426a397d |
     | service_name | placement                        |
     | service_type | placement                        |
     | url          | http://controller01:8778         |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne placement admin http://controller01:8778
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 06b3b3e4bb474be7b0381fd35c353ece |
     | interface    | admin                            |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 5ce8d1adb0d9468d8cbd6585426a397d |
     | service_name | placement                        |
     | service_type | placement                        |
     | url          | http://controller01:8778         |
     +--------------+----------------------------------+
     ```

  4. nova 组件安装

     软件包

     ```
     # yum install openstack-nova-api openstack-nova-conductor \
     openstack-nova-console openstack-nova-novncproxy \
     openstack-nova-scheduler openstack-nova-placement-api -y
     ```

  5. 配置文件

     ```
     # vim /etc/nova/nova.conf
     [DEFAULT]
     ...
     enabled_apis=osapi_compute,metadata
     ```

     数据库访问

     ```
     [api_database]
     ...
     connection=mysql+pymysql://nova:qwe123@controller01/nova_api
     ...
     [database]
     ...
     connection=mysql+pymysql://nova:qwe123@controller01/nova
     ```

     访问 RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url=rabbit://openstack:qwe123@controller01
     ```

     访问 Keystone

     ```
     [api]
     ...
     auth_strategy=keystone
     ...
     [keystone_authtoken]
     ...
     auth_uri=http://controller01:5000
     auth_url=http://controller01:35357
     memcached_servers=controller01:11211
     auth_type=password
     project_domain_name=default
     user_domain_name=default
     project_name=service
     username=nova
     password=qwe123
     ```

     OpenStack 网络包含防火墙功能，禁用系统内置防火墙

     ```
     [DEFAULT]
     ...
     use_neutron = True
     firewall_driver = nova.virt.firewall.NoopFirewallDriver
     ```

     VNC

     ```
     [vnc]
     enabled=true
     ...
     vncserver_listen=$my_ip
     vncserver_proxyclient_address=$my_ip
     ```

     Glance

     ```
     [glance]
     ...
     api_servers=http://controller01:9292
     ```

     oslo_concurrency

     ```
     [oslo_concurrency]
     ...
     lock_path=/var/lib/nova/tmp
     ```

     Placement API

     ```
     [placement]
     ...
     os_region_name=RegionOne
     project_domain_name=Default
     project_name=service
     auth_type=password
     user_domain_name=Default
     auth_url=http://controller01:35357/v3
     username=placement
     password=qwe123
     ```

     ```
     # /etc/httpd/conf.d/00-nova-placement-api.conf
     ...
     <Directory /usr/bin>
        <IfVersion >= 2.4>
          Require all granted
        </IfVersion>
        <IfVersion < 2.4>
          Order allow,deny
          Allow from all
        </IfVersion>
     </Directory>
     ```

     ```
     # systemctl restart httpd
     ```

  6. 初始化相关数据库

     ```
     # sh -c "nova-manage api_db sync" nova

     # sh -c "nova-manage cell_v2 map_cell0" nova

     # sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
     cc7b93f6-3c15-4589-854d-5dddf66e7f5c

     # sh -c "nova-manage db sync" nova
     ```

  7. 确认 nova 的 cell0 cell1 注册信息

     ```
     # nova-manage cell_v2 list_cells
     +-------+--------------------------------------+--------------------------------------+---------------------------------------------------+
     |  Name |                 UUID                 |            Transport URL             |                Database Connection                |
     +-------+--------------------------------------+--------------------------------------+---------------------------------------------------+
     | cell0 | 00000000-0000-0000-0000-000000000000 |                none:/                | mysql+pymysql://nova:****@controller01/nova_cell0 |
     | cell1 | cc7b93f6-3c15-4589-854d-5dddf66e7f5c | rabbit://openstack:****@controller01 |    mysql+pymysql://nova:****@controller01/nova    |
     +-------+--------------------------------------+--------------------------------------+---------------------------------------------------+
     ```

  8. 启动服务

     ```
     # systemctl enable openstack-nova-api.service \
     openstack-nova-consoleauth.service \
     openstack-nova-scheduler.service \
     openstack-nova-conductor.service \
     openstack-nova-novncproxy.service

     # systemctl start openstack-nova-api.service \
     openstack-nova-consoleauth.service \
     openstack-nova-scheduler.service \
     openstack-nova-conductor.service \
     openstack-nova-novncproxy.service
     ```

- 验证

  1. 服务组件

     ```
     # openstack compute service list
     +----+------------------+--------------+----------+---------+-------+----------------------------+
     | ID | Binary           | Host         | Zone     | Status  | State | Updated At                 |
     +----+------------------+--------------+----------+---------+-------+----------------------------+
     |  1 | nova-consoleauth | controller01 | internal | enabled | up    | 2019-01-23T06:03:17.000000 |
     |  2 | nova-scheduler   | controller01 | internal | enabled | up    | 2019-01-23T06:03:19.000000 |
     |  3 | nova-conductor   | controller01 | internal | enabled | up    | 2019-01-23T06:03:11.000000 |
     +----+------------------+--------------+----------+---------+-------+----------------------------+
     ```

  2. 验证 Identity 服务中 API 连接

     ```
     # openstack catalog list
     +-----------+-----------+-------------------------------------------+
     | Name      | Type      | Endpoints                                 |
     +-----------+-----------+-------------------------------------------+
     | keystone  | identity  | RegionOne                                 |
     |           |           |   public: http://controller01:5000/v3/    |
     |           |           | RegionOne                                 |
     |           |           |   internal: http://controller01:5000/v3/  |
     |           |           | RegionOne                                 |
     |           |           |   admin: http://controller01:35357/v3/    |
     |           |           |                                           |
     | nova      | compute   | RegionOne                                 |
     |           |           |   internal: http://controller01:8774/v2.1 |
     |           |           | RegionOne                                 |
     |           |           |   public: http://controller01:8774/v2.1   |
     |           |           | RegionOne                                 |
     |           |           |   admin: http://controller01:8774/v2.1    |
     |           |           |                                           |
     | placement | placement | RegionOne                                 |
     |           |           |   admin: http://controller01:8778         |
     |           |           | RegionOne                                 |
     |           |           |   public: http://controller01:8778        |
     |           |           | RegionOne                                 |
     |           |           |   internal: http://controller01:8778      |
     |           |           |                                           |
     | glance    | image     | RegionOne                                 |
     |           |           |   internal: http://controller01:9292      |
     |           |           | RegionOne                                 |
     |           |           |   public: http://controller01:9292        |
     |           |           | RegionOne                                 |
     |           |           |   admin: http://controller01:9292         |
     |           |           |                                           |
     +-----------+-----------+-------------------------------------------+
     ```

  3. 验证镜像服务连接

     ```
     # openstack image list
     +--------------------------------------+--------------+--------+
     | ID                                   | Name         | Status |
     +--------------------------------------+--------------+--------+
     | 3a295fc0-6899-4893-8390-e2217eab3542 | cirros-0.3.5 | active |
     +--------------------------------------+--------------+--------+
     ```

  4. 验证 cell 和 placement API

     ```
     # nova-status upgrade check
     +--------------------------------------------------------------------+
     | Upgrade Check Results                                              |
     +--------------------------------------------------------------------+
     | Check: Cells v2                                                    |
     | Result: Success                                                    |
     | Details: No host mappings or compute nodes were found. Remember to |
     |   run command 'nova-manage cell_v2 discover_hosts' when new        |
     |   compute hosts are deployed.                                      |
     +--------------------------------------------------------------------+
     | Check: Placement API                                               |
     | Result: Success                                                    |
     | Details: None                                                      |
     +--------------------------------------------------------------------+
     | Check: Resource Providers                                          |
     | Result: Success                                                    |
     | Details: There are no compute resource providers in the Placement  |
     |   service nor are there compute nodes in the database.             |
     |   Remember to configure new compute nodes to report into the       |
     |   Placement service. See                                           |
     |   http://docs.openstack.org/developer/nova/placement.html          |
     |   for more details.                                                |
     +--------------------------------------------------------------------+
     ```

     注：此时尚未添加计算节点

     启动计算节点后，加入 cell 数据库

     ```
     # sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
     Found 2 cell mappings.
     Skipping cell0 since it does not contain hosts.
     Getting computes from cell 'cell1': cc7b93f6-3c15-4589-854d-5dddf66e7f5c
     Checking host mapping for compute host 'computer01': 44552253-8b5a-4661-8f74-47a4dde134ef
     Creating host mapping for compute host 'computer01': 44552253-8b5a-4661-8f74-47a4dde134ef
     Found 1 unmapped computes in cell: cc7b93f6-3c15-4589-854d-5dddf66e7f5c
     ```

     再次查看

     ```
     # openstack compute service list
     +----+------------------+--------------+----------+---------+-------+----------------------------+
     | ID | Binary           | Host         | Zone     | Status  | State | Updated At                 |
     +----+------------------+--------------+----------+---------+-------+----------------------------+
     |  1 | nova-consoleauth | controller01 | internal | enabled | up    | 2019-01-23T07:35:29.000000 |
     |  2 | nova-scheduler   | controller01 | internal | enabled | up    | 2019-01-23T07:35:29.000000 |
     |  3 | nova-conductor   | controller01 | internal | enabled | up    | 2019-01-23T07:35:32.000000 |
     |  6 | nova-compute     | computer01   | nova     | enabled | up    | 2019-01-23T07:35:29.000000 |
     +----+------------------+--------------+----------+---------+-------+----------------------------+

     # nova-status upgrade check
     +---------------------------+
     | Upgrade Check Results     |
     +---------------------------+
     | Check: Cells v2           |
     | Result: Success           |
     | Details: None             |
     +---------------------------+
     | Check: Placement API      |
     | Result: Success           |
     | Details: None             |
     +---------------------------+
     | Check: Resource Providers |
     | Result: Success           |
     | Details: None             |
     +---------------------------+
     ```

  5. 设置自动发现注册新增的计算节点

     ```
     # vim /etc/nova/nova.conf
     ...
     [scheduler]
     discover_hosts_in_cells_interval=300
     ```

### computer01

- 安装与配置

  1. 组件包

     ```
     # yum install openstack-nova-compute -y
     ```

  2. 配置文件

     判断计算节点是否支持硬件加速

     ```
     # egrep -c '(vmx|svm)' /proc/cpuinfo
     0
     ```

     因为是虚拟机，结果为 '0'，不支持硬件加速，使用 QEMU 替代 KVM

     ```
     # vim /etc/nova/nova.conf
     [libvirt]
     ...
     virt_type=qemu
     ```

     基本信息

     ```
     [DEFAULT]
     ...
     enabled_apis=osapi_compute,metadata
     ...
     my_ip=10.0.0.31
     ...
     use_neutron=true
     ...
     firewall_driver=nova.virt.firewall.NoopFirewallDriver
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url=rabbit://openstack:qwe123@controller01
     ```

     Keystone 对接

     ```
     [api]
     ...
     auth_strategy=keystone

     [keystone_authtoken]
     ...
     auth_uri=http://controller01:5000
     auth_url=http://controller01:35357
     memcached_servers=controller01:11211
     auth_type=password
     project_domain_name=default
     user_domain_name=default
     project_name=service
     username=nova
     password=qwe123
     ```

     VNC

     ```
     [vnc]
     ...
     enabled=True
     vncserver_listen=0.0.0.0
     vncserver_proxyclient_address=$my_ip
     novncproxy_base_url=http://controller01:6080/vnc_auto.html
     ```

     Glance 对接

     ```
     [glance]
     ...
     api_servers=http://controller01:9292
     ```

     oslo_concurrency

     ```
     [oslo_concurrency]
     ...
     lock_path = /var/lib/nova/tmp
     ```

     Placement API

     ```
     [placement]
     ...
     os_region_name=RegionOne
     project_domain_name=Default
     project_name=service
     auth_type=password
     user_domain_name=Default
     auth_url=http://controller01:35357/v3
     username=placement
     password=qwe123
     ```

  3. 启动服务

     ```
     # systemctl enable libvirtd.service openstack-nova-compute.service

     # systemctl start libvirtd.service openstack-nova-compute.service
     ```

     注：需要打开 RabbitMQ 所在节点的 5672 端口

- 验证

  见控制节点验证最后部分
