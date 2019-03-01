---
title: "OpenStack Pike On CentOS7 手动部署（下）"
categories:
  - 云计算
date: 2019-2-12
toc: true
toc_label: "部署过程"
toc_icon: "align-left"
header:
  teaser: /assets/images/openstacklogo.jpeg
---

> Ceilometer & Heat

## Ceilometer

### computer01

- 安装与配置

  1. 组件

     ```
     # yum install openstack-ceilometer-compute -y
     ```

  2. 配置文件

     ```
     # vim /etc/ceilometer/ceilometer.conf
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     凭证

     ```
     [service_credentials]
     ...
     auth_url = http://controller01:5000
     project_domain_id = default
     user_domain_id = default
     auth_type = password
     username = ceilometer
     project_name = service
     password = qwe123
     interface = internalURL
     region_name = RegionOne
     ```

  3. 配置计算服务使用 Telemetry

     ```
     # vim /etc/nova/nova.conf
     [DEFAULT]
     ...
     instance_usage_audit = True
     instance_usage_audit_period = hour
     notify_on_state_change = vm_and_task_state
     ...
     [oslo_messaging_notifications]
     ...
     driver = messagingv2
     ```

- 服务

  1. ceilometer

     ```
     # systemctl enable openstack-ceilometer-compute.service

     # systemctl start openstack-ceilometer-compute.service
     ```

  2. 重启计算服务

     ```
     # systemctl restart openstack-nova-compute.service
     ```

### controller01

使用 Gnocchi 作为计量数据的存储后端

- 安装与配置

  1. ceilometer 凭证

     ```
     # . admin_openrc
     ```

     创建 ceilometer 用户

     ```
     # openstack user create --domain default --password-prompt ceilometer
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | f6f0919a4dac4044a3e4fccbeeb85be3 |
     | name                | ceilometer                       |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+
     ```

     赋权

     ```
     # openstack role add --project service --user ceilometer admin
     ```

     创建 ceilometer 服务实体

     ```
     # openstack service create --name ceilometer --description "Telemetry" metering
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | Telemetry                        |
     | enabled     | True                             |
     | id          | 33a5d349182241a5b7b875c3bc028422 |
     | name        | ceilometer                       |
     | type        | metering                         |
     +-------------+----------------------------------+
     ```

  2. Gnocchi 凭证

     创建 gnocchi 用户

     ```
     # openstack user create --domain default --password-prompt gnocchi
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | 738c384cbb4f4b2a8dcc56d6a13ffd9d |
     | name                | gnocchi                          |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+
     ```

     创建 gnocchi 服务实体

     ```
     # openstack service create --name gnocchi --description "Metric Service" metric
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | Metric Service                   |
     | enabled     | True                             |
     | id          | 710901e78f704ff7aad17913e8285c39 |
     | name        | gnocchi                          |
     | type        | metric                           |
     +-------------+----------------------------------+
     ```

     赋权

     ```
     # openstack role add --project service --user gnocchi admin
     ```

  3. 创建 Metric 服务 API endpoints

     ```
     # openstack endpoint create --region RegionOne metric public http://controller01:8041
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 15894d2a6f72485f9db801dadc691219 |
     | interface    | public                           |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 710901e78f704ff7aad17913e8285c39 |
     | service_name | gnocchi                          |
     | service_type | metric                           |
     | url          | http://controller01:8041         |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne metric internal http://controller01:8041
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 70d1d951601d433989c91442faca48e0 |
     | interface    | internal                         |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 710901e78f704ff7aad17913e8285c39 |
     | service_name | gnocchi                          |
     | service_type | metric                           |
     | url          | http://controller01:8041         |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne metric admin http://controller01:8041
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | b27ffe3e56a14216ad58e34551573e5c |
     | interface    | admin                            |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 710901e78f704ff7aad17913e8285c39 |
     | service_name | gnocchi                          |
     | service_type | metric                           |
     | url          | http://controller01:8041         |
     +--------------+----------------------------------+
     ```

  4. 安装 Gnocchi

     ```
     # yum install redis openstack-gnocchi-api openstack-gnocchi-metricd python-gnocchiclient -y
     ```

     数据库

     ```
     MariaDB [(none)]> CREATE DATABASE gnocchi;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.01 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> quit
     Bye
     ```

     配置文件

     ```
     # vim /etc/gnocchi/gnocchi.conf
     ...
     [api]
     auth_mode = keystone
     ...
     [keystone_authtoken]
     ...
     auth_url = http://controller01:5000
     auth_url = http://controller01:35357
     memcached_servers = controller01:11211
     auth_type = password
     project_domain_name = default
     user_domain_name = default
     project_name = service
     username = gnocchi
     password = qwe123
     interface = internalURL
     region_name = RegionOne
     service_token_roles_required = true
     ...
     [indexer]
     url = mysql+pymysql://gnocchi:qwe123@controller01/gnocchi
     ...
     [storage]
     file_basepath = /var/lib/gnocchi
     driver = file
     ```

     初始化

     ```
     # gnocchi-upgrade
     ```

     修改 gnocchi-api 端口为 8041

     ```
     # vim /usr/bin/gnocchi-api
     ...
     #parser.add_argument('--port', '-p', type=int, default=8000,
     parser.add_argument('--port', '-p', type=int, default=8041,
     ```

     服务

     ```
     # systemctl enable openstack-gnocchi-api.service openstack-gnocchi-metricd.service

     # systemctl start openstack-gnocchi-api.service openstack-gnocchi-metricd.service
     ```

  5. Ceilometer 组件

     ```
     # yum install openstack-ceilometer-notification openstack-ceilometer-central -y
     ```

  6. 配置文件

     ```
     # vim /etc/ceilometer/ceilometer.conf
     ```

     Gnocchi 连接

     ```
     [dispatcher_gnocchi]
     ...
     filter_service_activity = False
     archive_policy = low
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     凭证

     ```
     [service_credentials]
     ...
     auth_url = http://controller01:5000
     auth_url = http://controller01:35357
     memcached_servers = controller01:11211
     auth_type = password
     project_domain_id = default
     user_domain_id = default
     project_name = service
     username = ceilometer
     password = qwe123
     interface = internalURL
     region_name = RegionOne
     ```

  7. 在 Gnocchi 里创建 ceilometer 资源

     ```
     # ceilometer-upgrade --skip-metering-database
     ```

- 服务

  1. Telemetry services

     ```
     # systemctl enable openstack-ceilometer-notification.service openstack-ceilometer-central.service

     # systemctl start openstack-ceilometer-notification.service openstack-ceilometer-central.service
     ```

## Ceilometer 对接 Cinder

### controller01

- 配置文件

  1. 配置 notifications

     ```
     # vim /etc/cinder/cinder.conf
     [oslo_messaging_notifications]
     ...
     driver = messagingv2
     ```

- 重启服务

  1. Block Storage services

     ```
     # systemctl restart openstack-cinder-api.service openstack-cinder-scheduler.service
     ```

### computer01

- 重启服务

  1. Block Storage services

     ```
     # systemctl restart openstack-cinder-volume.service
     ```

## Ceilometer 对接 Glance

### controller01

- 配置文件

  1. api

     ```
     # vim /etc/glance/glance-api.conf
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ...
     [oslo_messaging_notifications]
     ...
     driver = messagingv2
     ```

  2. registry

     ```
     # vim /etc/glance/glance-registry.conf
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ...
     [oslo_messaging_notifications]
     ...
     driver = messagingv2
     ```

- 重启服务

  1. api & registry

     ```
     # systemctl restart openstack-glance-api.service openstack-glance-registry.service
     ```

- 验证

  1. 下载镜像

     ```
     # wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
     ```

  2. 上传镜像

     ```
     # openstack image create "cirros-0.4.0" \
     --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 \
     --container-format bare --public
     ```

  3. 查看记录

     ```
     # gnocchi resource list --type  image
     +--------------------------------------+-------+----------------------------------+---------+--------------------------------------+----------------------------------+----------+----------------------------------+--------------+-------------------------------------------------------------------+------------------+-------------+--------------+
     | id                                   | type  | project_id                       | user_id | original_resource_id                 | started_at                       | ended_at | revision_start                   | revision_end | creator                                                           | container_format | disk_format | name         |
     +--------------------------------------+-------+----------------------------------+---------+--------------------------------------+----------------------------------+----------+----------------------------------+--------------+-------------------------------------------------------------------+------------------+-------------+--------------+
     | 91d84be9-540d-48bd-9555-f7f337df60de | image | 0a7aef2ccc8b4a71b80534b7c43fe8ab | None    | 91d84be9-540d-48bd-9555-f7f337df60de | 2019-02-18T06:38:38.468978+00:00 | None     | 2019-02-18T06:38:38.469085+00:00 | None         | f6f0919a4dac4044a3e4fccbeeb85be3:5f81944c65c346efa9ff8e0753ba795f | bare             | qcow2       | cirros-0.4.0 |
     +--------------------------------------+-------+----------------------------------+---------+--------------------------------------+----------------------------------+----------+----------------------------------+--------------+-------------------------------------------------------------------+------------------+-------------+--------------+

     # gnocchi resource show 91d84be9-540d-48bd-9555-f7f337df60de
     +-----------------------+-------------------------------------------------------------------+
     | Field                 | Value                                                             |
     +-----------------------+-------------------------------------------------------------------+
     | created_by_project_id | 5f81944c65c346efa9ff8e0753ba795f                                  |
     | created_by_user_id    | f6f0919a4dac4044a3e4fccbeeb85be3                                  |
     | creator               | f6f0919a4dac4044a3e4fccbeeb85be3:5f81944c65c346efa9ff8e0753ba795f |
     | ended_at              | None                                                              |
     | id                    | 91d84be9-540d-48bd-9555-f7f337df60de                              |
     | metrics               | image.download: 01914954-e679-4821-b665-0a5f5cbfae88              |
     |                       | image.serve: 3e7cbfa4-9b71-4c42-aec4-017d03203d68                 |
     |                       | image.size: d9e359a1-0b5e-46e6-a2fd-9fa54f1a1eca                  |
     | original_resource_id  | 91d84be9-540d-48bd-9555-f7f337df60de                              |
     | project_id            | 0a7aef2ccc8b4a71b80534b7c43fe8ab                                  |
     | revision_end          | None                                                              |
     | revision_start        | 2019-02-18T06:38:38.469085+00:00                                  |
     | started_at            | 2019-02-18T06:38:38.468978+00:00                                  |
     | type                  | image                                                             |
     | user_id               | None                                                              |
     +-----------------------+-------------------------------------------------------------------+
     ```

## Ceilometer 对接 Neutron

### controller01

- 配置文件

  1. 配置 notifications

     ```
     # vim /etc/neutron/neutron.conf
     [oslo_messaging_notifications]
     ...
     driver = messagingv2
     ```

- 重启服务

  1. neutron 服务

     ```
     # systemctl restart neutron-server.service
     ```


## Heat

### controller01

- 安装与配置

  1. 数据库

     ```
     MariaDB [(none)]> CREATE DATABASE heat;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> quit
     Bye
     ```

  2. 凭证

     创建 heat 用户

     ```
     # openstack user create --domain default --password-prompt heat
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | 88947055e5b04b87b0b9c5439302c2db |
     | name                | heat                             |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+
     ```

     赋权

     ```
     # openstack role add --project service --user heat admin
     ```

     创建 heat 服务实体

     ```
     # openstack service create --name heat --description "Orchestration" orchestration
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | Orchestration                    |
     | enabled     | True                             |
     | id          | 8dc1a6a34c0644e69d640838cf01ad2d |
     | name        | heat                             |
     | type        | orchestration                    |
     +-------------+----------------------------------+
     ```

     创建 heat-cfn 服务实体

     ```
     # openstack service create --name heat-cfn --description "Orchestration"  cloudformation
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | Orchestration                    |
     | enabled     | True                             |
     | id          | 2a43f9554521456797c11e646c94e88c |
     | name        | heat-cfn                         |
     | type        | cloudformation                   |
     +-------------+----------------------------------+
     ```

  3. 创建 Orchestration 服务 API endpoints

     ```
     # openstack endpoint create --region RegionOne orchestration public http://controller01:8004/v1/%\(tenant_id\)s
     +--------------+-------------------------------------------+
     | Field        | Value                                     |
     +--------------+-------------------------------------------+
     | enabled      | True                                      |
     | id           | 9ae61376525947e8bb14c14d4534696c          |
     | interface    | public                                    |
     | region       | RegionOne                                 |
     | region_id    | RegionOne                                 |
     | service_id   | 8dc1a6a34c0644e69d640838cf01ad2d          |
     | service_name | heat                                      |
     | service_type | orchestration                             |
     | url          | http://controller01:8004/v1/%(tenant_id)s |
     +--------------+-------------------------------------------+

     # openstack endpoint create --region RegionOne orchestration internal http://controller01:8004/v1/%\(tenant_id\)s
     +--------------+-------------------------------------------+
     | Field        | Value                                     |
     +--------------+-------------------------------------------+
     | enabled      | True                                      |
     | id           | 7a948b88f4a74df8b2a2e9504dcf7820          |
     | interface    | internal                                  |
     | region       | RegionOne                                 |
     | region_id    | RegionOne                                 |
     | service_id   | 8dc1a6a34c0644e69d640838cf01ad2d          |
     | service_name | heat                                      |
     | service_type | orchestration                             |
     | url          | http://controller01:8004/v1/%(tenant_id)s |
     +--------------+-------------------------------------------+

     # openstack endpoint create --region RegionOne orchestration admin http://controller01:8004/v1/%\(tenant_id\)s
     +--------------+-------------------------------------------+
     | Field        | Value                                     |
     +--------------+-------------------------------------------+
     | enabled      | True                                      |
     | id           | 1775361dedf64ff087c82dfee67479f9          |
     | interface    | admin                                     |
     | region       | RegionOne                                 |
     | region_id    | RegionOne                                 |
     | service_id   | 8dc1a6a34c0644e69d640838cf01ad2d          |
     | service_name | heat                                      |
     | service_type | orchestration                             |
     | url          | http://controller01:8004/v1/%(tenant_id)s |
     +--------------+-------------------------------------------+

     # openstack endpoint create --region RegionOne cloudformation public http://controller01:8000/v1
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | c9bf7fe607fb43f99bc3a542d0871e7a |
     | interface    | public                           |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 2a43f9554521456797c11e646c94e88c |
     | service_name | heat-cfn                         |
     | service_type | cloudformation                   |
     | url          | http://controller01:8000/v1      |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne cloudformation internal http://controller01:8000/v1
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | e2936e623038446baca40355531ae757 |
     | interface    | internal                         |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 2a43f9554521456797c11e646c94e88c |
     | service_name | heat-cfn                         |
     | service_type | cloudformation                   |
     | url          | http://controller01:8000/v1      |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne cloudformation admin http://controller01:8000/v1
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | c42c8b4074e94d4096725841d236df64 |
     | interface    | admin                            |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 2a43f9554521456797c11e646c94e88c |
     | service_name | heat-cfn                         |
     | service_type | cloudformation                   |
     | url          | http://controller01:8000/v1      |
     +--------------+----------------------------------+
     ```

  4. Orchestration 服务补充认证信息

     包含 stack 项目和用户的 heat 域

     ```
     # openstack domain create --description "Stack projects and users" heat
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | Stack projects and users         |
     | enabled     | True                             |
     | id          | df153a5243b44b69a3fab5ed776c3429 |
     | name        | heat                             |
     +-------------+----------------------------------+
     ```

     在 heat 域中创建管理项目和用户的  heat_domain_admin 用户

     ```
     # openstack user create --domain heat --password-prompt heat_domain_admin
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | df153a5243b44b69a3fab5ed776c3429 |
     | enabled             | True                             |
     | id                  | 73adb5b77b0546f3864e223e73053629 |
     | name                | heat_domain_admin                |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+
     ```

     赋权

     ```
     # openstack role add --domain heat --user-domain heat --user heat_domain_admin admin
     ```

     创建 heat_stack_owner 角色

     ```
     # openstack role create heat_stack_owner
     +-----------+----------------------------------+
     | Field     | Value                            |
     +-----------+----------------------------------+
     | domain_id | None                             |
     | id        | 370860e1ef6e4077a76946e6e52ec958 |
     | name      | heat_stack_owner                 |
     +-----------+----------------------------------+
     ```

     赋权

     ```
     # openstack role add --project demo --user demo heat_stack_owner
     ```

     创建 heat_stack_user 角色

     ```
     # openstack role create heat_stack_user
     +-----------+----------------------------------+
     | Field     | Value                            |
     +-----------+----------------------------------+
     | domain_id | None                             |
     | id        | 9986868b17cf4035bca7b9b9dd1dd49e |
     | name      | heat_stack_user                  |
     +-----------+----------------------------------+
     ```

  5. 安装组件

     ```
     # yum install openstack-heat-api openstack-heat-api-cfn openstack-heat-engine -y
     ```

  6. 配置文件

     ```
     # vim /etc/heat/heat.conf
     ```

     数据库

     ```
     [database]
     ...
     connection = mysql+pymysql://heat:qwe123@controller01/heat
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     认证信息

     ```
     [keystone_authtoken]
     ...
     auth_uri = http://controller01:5000
     auth_url = http://controller01:35357
     memcached_servers = controller01:11211
     auth_type = password
     project_domain_name = default
     user_domain_name = default
     project_name = service
     username = heat
     password = qwe123
     ...
     [trustee]
     ...
     auth_type = password
     auth_url = http://controller01:35357
     username = heat
     password = qwe123
     user_domain_name = default
     ...
     [clients_keystone]
     ...
     auth_uri = http://controller01:35357
     ...
     [ec2authtoken]
     ...
     auth_uri = http://controller01:5000/v3
     ```

     metadata & wait condition URLs

     ```
     [DEFAULT]
     ...
     heat_metadata_server_url = http://controller01:8000
     heat_waitcondition_server_url = http://controller01:8000/v1/waitcondition
     ```

     栈域与管理凭据

     ```
     [DEFAULT]
     ...
     stack_domain_admin = heat_domain_admin
     stack_domain_admin_password = qwe123
     stack_user_domain_name = heat
     ```

  7. 同步 Orchestration 数据库

     ```
     # sh -c "heat-manage db_sync" heat
     ```

- 启动服务

     ```
     # systemctl enable openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service

     # systemctl start openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
     ```

- 验证

  是否注册了每个 cpu 进程

     ```
     # . admin_openrc

     # openstack orchestration service list
     +--------------+-------------+--------------------------------------+--------------+--------+----------------------------+--------+
     | Hostname     | Binary      | Engine ID                            | Host         | Topic  | Updated At                 | Status |
     +--------------+-------------+--------------------------------------+--------------+--------+----------------------------+--------+
     | controller01 | heat-engine | 8347a222-b58a-4218-9b4d-d0a89d790c2a | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | 09e041ca-e9bc-4faf-8476-a489bd9dc1f2 | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | 1ae6e13c-1439-4659-b846-60107c5deefb | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | 050b46ba-fbd5-4104-8d19-59cc621e6cc3 | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | a73d80ec-60e9-40ab-8762-a84759f19be3 | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | 220fe081-d77e-42a2-b96b-ab93a77fc2aa | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | 8b1731ca-ba98-423a-8325-85f7ddc9262d | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     | controller01 | heat-engine | 88e956a9-7764-4c3b-a128-8a65da7f88bc | controller01 | engine | 2019-02-14T02:15:18.000000 | up     |
     +--------------+-------------+--------------------------------------+--------------+--------+----------------------------+--------+
     ```

- 测试实例

  1. 创建模板

     ```
     # vim demo-template.yml
     heat_template_version: 2019-02-14
     description: Launch a basic instance with CirrOS-0.3.5 image using the
                  ``testflavor01`` flavor, ``demokey01`` key,  and one network.

     parameters:
       NetID:
         type: string
         description: Network ID to use for the instance.

     resources:
       server:
         type: OS::Nova::Server
         properties:
           image: cirros-0.3.5
           flavor: testflavor01
           key_name: demokey01
           networks:
           - network: { get_param: NetID }

     outputs:
       instance_name:
         description: Name of the instance.
         value: { get_attr: [ server, name ] }
       instance_ip:
         description: IP address of the instance.
         value: { get_attr: [ server, first_address ] }
     ```

  2. 创建一个栈（stack）

     普通用户模式

     ```
     # . demo_openrc
     ```

     查看可用网络

     ```
     # openstack network list
     +--------------------------------------+-----------+--------------------------------------+
     | ID                                   | Name      | Subnets                              |
     +--------------------------------------+-----------+--------------------------------------+
     | ec70d7b2-6219-4be0-89bb-728709e8a716 | testnet01 | c9e7eee7-9aed-4b65-b373-ba8d01b991b7 |
     +--------------------------------------+-----------+--------------------------------------+
     ```

     获取网络 ID

     ```
     # export NET_ID=$(openstack network list | awk '/ testnet01 / { print $2 }')
     ```

     创建栈

     ```
     # openstack stack create -t demo-template.yml --parameter "NetID=$NET_ID" stack
     +---------------------+-------------------------------------------------------------------------------------------------------------------+
     | Field               | Value                                                                                                             |
     +---------------------+-------------------------------------------------------------------------------------------------------------------+
     | id                  | a3230c90-c2c6-4cb0-9d4f-474b61ae0b83                                                                              |
     | stack_name          | stack                                                                                                             |
     | description         | Launch a basic instance with CirrOS image using the ``testflavor01`` flavor, ``demokey01`` key,  and one network. |
     | creation_time       | 2019-02-14T03:23:33Z                                                                                              |
     | updated_time        | None                                                                                                              |
     | stack_status        | CREATE_IN_PROGRESS                                                                                                |
     | stack_status_reason | Stack CREATE started                                                                                              |
     +---------------------+-------------------------------------------------------------------------------------------------------------------+
     ```

  3. 查看

     ```
     # openstack stack list
     +--------------------------------------+------------+----------------+----------------------+--------------+
     | ID                                   | Stack Name | Stack Status   | Creation Time        | Updated Time |
     +--------------------------------------+------------+----------------+----------------------+--------------+
     | a3230c90-c2c6-4cb0-9d4f-474b61ae0b83 | stack      | CHECK_COMPLETE | 2019-02-14T03:23:33Z | None         |
     +--------------------------------------+------------+----------------+----------------------+--------------+

     # openstack stack output show --all stack
     +---------------+-------------------------------------------------+
     | Field         | Value                                           |
     +---------------+-------------------------------------------------+
     | instance_name | {                                               |
     |               |   "output_value": "stack-server-hhslficzotj3",  |
     |               |   "output_key": "instance_name",                |
     |               |   "description": "Name of the instance."        |
     |               | }                                               |
     | instance_ip   | {                                               |
     |               |   "output_value": "192.168.1.8",                |
     |               |   "output_key": "instance_ip",                  |
     |               |   "description": "IP address of the instance."  |
     |               | }                                               |
     +---------------+-------------------------------------------------+

     # openstack server list
     +--------------------------------------+---------------------------+--------+-----------------------+--------------+--------------+
     | ID                                   | Name                      | Status | Networks              | Image        | Flavor       |
     +--------------------------------------+---------------------------+--------+-----------------------+--------------+--------------+
     | 719829e4-19be-41e6-93a8-6a87ce512fad | stack-server-hhslficzotj3 | ACTIVE | testnet01=192.168.1.8 | cirros-0.3.5 | testflavor01 |
     +--------------------------------------+---------------------------+--------+-----------------------+--------------+--------------+
     ```
