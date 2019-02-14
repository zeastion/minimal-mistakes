---
title: "OpenStack Pike On CentOS7 手动部署（中）"
categories:
  - 云计算
date: 2019-1-24
toc: true
toc_label: "部署过程"
toc_icon: "align-left"
header:
  teaser: /assets/images/openstacklogo.jpeg
---

> Neutron & Dashboard & Cinder

## Neutron

注：选择第二种方式 “Self-service networks”

### controller01

- 安装与配置

  1. 数据库

     ```
     # mysql -uroot -p
     MariaDB [(none)]> CREATE DATABASE neutron;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> quit
     Bye
     ```

  2. neutron 用户相关设置

     创建 neutron 用户并赋权

     ```
     # openstack user create --domain default --password-prompt neutron
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | d58cd351c74c43d68cc4b98820d4f5b2 |
     | name                | neutron                          |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+

     # openstack role add --project service --user neutron admin
     ```

     创建 neutron 服务实体

     ```
     # openstack service create --name neutron --description "OpenStack Networking" network
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | OpenStack Networking             |
     | enabled     | True                             |
     | id          | 6aed8bf289ce47e5974f38c5810d3bf0 |
     | name        | neutron                          |
     | type        | network                          |
     +-------------+----------------------------------+
     ```

     创建 Networking service 的 API endpoints

     ```
     # openstack endpoint create --region RegionOne network public http://controller01:9696
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 72980bd9a36e4b36a27b19ecee185742 |
     | interface    | public                           |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 6aed8bf289ce47e5974f38c5810d3bf0 |
     | service_name | neutron                          |
     | service_type | network                          |
     | url          | http://controller01:9696         |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne network internal http://controller01:9696
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 793f845e39af489fb9a2285f90535312 |
     | interface    | internal                         |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 6aed8bf289ce47e5974f38c5810d3bf0 |
     | service_name | neutron                          |
     | service_type | network                          |
     | url          | http://controller01:9696         |
     +--------------+----------------------------------+

     # openstack endpoint create --region RegionOne network admin http://controller01:9696
     +--------------+----------------------------------+
     | Field        | Value                            |
     +--------------+----------------------------------+
     | enabled      | True                             |
     | id           | 2ecbf22a2bd542e68ebb7ca51b19b63c |
     | interface    | admin                            |
     | region       | RegionOne                        |
     | region_id    | RegionOne                        |
     | service_id   | 6aed8bf289ce47e5974f38c5810d3bf0 |
     | service_name | neutron                          |
     | service_type | network                          |
     | url          | http://controller01:9696         |
     +--------------+----------------------------------+
     ```

  3. neutron 组件安装

     ```
     # yum install openstack-neutron openstack-neutron-ml2 \
     openstack-neutron-linuxbridge ebtables -y
     ```

  4. neutron 配置文件

     ```
     # vim /etc/neutron/neutron.conf
     ```

     数据库连接

     ```
     [database]
     ...
     connection = mysql+pymysql://neutron:qwe123@controller01/neutron
     ```

     启用Modular Layer 2 (ML2)插件，路由服务和重叠的 IP 地址

     ```
     [DEFAULT]
     ...
     core_plugin = ml2
     service_plugins = router
     allow_overlapping_ips = True
     ```

     RabbitMQ 连接

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     Identity service 访问

     ```
     [DEFAULT]
     ...
     auth_strategy = keystone
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
     username = neutron
     password = qwe123
     ```

     配置网络服务来通知 nova 网络拓扑更改

     ```
     [DEFAULT]
     ...
     notify_nova_on_port_status_changes = true
     notify_nova_on_port_data_changes = true

     [nova]
     ...
     auth_url = http://controller01:35357
     auth_type = password
     project_domain_name = default
     user_domain_name = default
     region_name = RegionOne
     project_name = service
     username = nova
     password = qwe123
     ```

     配置锁路径

     ```
     [oslo_concurrency]
     ...
     lock_path = /var/lib/neutron/tmp
     ```

  5. 配置 Modular Layer 2 插件

     ML2 插件使用 Linuxbridge 机制为实例创建 layer-2 虚拟网络

     ```
     # vim /etc/neutron/plugins/ml2/ml2_conf.ini
     ```

     启用 flat，VLAN 及 VXLAN 网络

     ```
     [ml2]
     ...
     type_drivers = flat,vlan,vxlan
     ```

     启用 VXLAN 私有网络

     ```
     [ml2]
     ...
     tenant_network_types = vxlan
     ```

     启用 Linuxbridge 和 layer-2 population

     注： Linuxbridge 代理只支持 VXLAN overlay 网络

     ```
     [ml2]
     ...
     mechanism_drivers = linuxbridge,l2population
     ```

     启用端口安全扩展驱动

     ```
     [ml2]
     ...
     extension_drivers = port_security
     ```

     配置公共虚拟网络 (provider virtual network) 为 flat 网络

     ```
     [ml2_type_flat]
     ...
     flat_networks = provider
     ```

     为私有网络 (self-service networks) 配置 VXLAN 网络识别的网络范围

     ```
     [ml2_type_vxlan]
     ...
     vni_ranges = 1:1000
     ```

     启用 ipset 增加安全组规则的高效性

     ```
     [securitygroup]
     ...
     enable_ipset = true
     ```

  6. 配置 Linux bridge 代理

     Linuxbridge 代理为实例建立 layer-2 (bridging and switching) 虚拟网络并且处理安全组规则

     ```
     # vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
     ```

     将公共虚拟网络和公共物理网络接口对应起来

     此处 'PROVIDER_INTERFACE_NAME' 即 controller 节点的 Provider network 网口 (ens33)

     ```
     [linux_bridge]
     physical_interface_mappings = provider:ens33
     ```

     启用 VXLAN overlay 网络

     配置 overlay 网络的物理网络接口IP地址，即Management network IP (10.0.0.11)

     启用 layer-2 population

     ```
     [vxlan]
     enable_vxlan = true
     local_ip = 10.0.0.11
     l2_population = true
     ```

     安全组及防火墙

     ```
     [securitygroup]
     ...
     enable_security_group = true
     firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
     ```

  7. 配置 layer-3 代理

     Layer-3 代理为私有虚拟网络提供路由和 NAT 服务

     ```
     # vim /etc/neutron/l3_agent.ini
     ```

     配置 Linuxbridge 接口驱动和外部网络网桥

  8. 配置 DHCP 代理

     ```
     # vim /etc/neutron/dhcp_agent.ini
     ```

     配置 Linuxbridge 驱动接口，DHCP 驱动

     启用隔离元数据，这样在 provider networks 上的实例就可以通过网络来访问元数据

     ```
     [DEFAULT]
     ...
     interface_driver = linuxbridge
     dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
     enable_isolated_metadata = true
     ```

  9. 配置元数据代理

     配置元数据主机以及共享密码

     ```
     # vim /etc/neutron/metadata_agent.ini
     [DEFAULT]
     ...
     nova_metadata_host = controller01
     metadata_proxy_shared_secret = qwe123
     ```

  10. 计算服务对接 Neutron

      ```
      # vim /etc/nova/nova.conf
      ...
      [neutron]
      ...
      url = http://controller01:9696
      auth_url = http://controller01:35357
      auth_type = password
      project_domain_name = default
      user_domain_name = default
      region_name = RegionOne
      project_name = service
      username = neutron
      password = qwe123
      service_metadata_proxy = true
      metadata_proxy_shared_secret = qwe123
      ```

  11. 超链接

      ```
      # ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
      ```

  12. 填充数据库

      ```
      # sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
      --config-file /etc/neutron/plugins/ml2/ml2_conf.ini \
      upgrade head" neutron
      ```

  13. 重启 nova-api

      ```
      # systemctl restart openstack-nova-api.service
      ```

- 启动服务

      ```
      # systemctl enable neutron-server.service \
      neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
      neutron-metadata-agent.service

      # systemctl start neutron-server.service \
      neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
      neutron-metadata-agent.service

      # systemctl enable neutron-l3-agent.service

      # systemctl start neutron-l3-agent.service
      ```

- 验证

  1. 验证 neutron-server 服务是否正常

     ```
     # openstack extension list --network
     +----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
     | Name                                                                                         | Alias                     | Description                                                                                                                                              |
     +----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
     | Default Subnetpools                                                                          | default-subnetpools       | Provides ability to mark and use a subnetpool as the default                                                                                             |
     | Network IP Availability                                                                      | network-ip-availability   | Provides IP availability data for each network and subnet.                                                                                               |
     | Network Availability Zone                                                                    | network_availability_zone | Availability zone support for network.                                                                                                                   |
     | Auto Allocated Topology Services                                                             | auto-allocated-topology   | Auto Allocated Topology Services.                                                                                                                        |
     | Neutron L3 Configurable external gateway mode                                                | ext-gw-mode               | Extension of the router abstraction for specifying whether SNAT should occur on the external gateway                                                     |
     | Port Binding                                                                                 | binding                   | Expose port bindings of a virtual port to external application                                                                                           |
     | agent                                                                                        | agent                     | The agent management extension.                                                                                                                          |
     | Subnet Allocation                                                                            | subnet_allocation         | Enables allocation of subnets from a subnet pool                                                                                                         |
     | L3 Agent Scheduler                                                                           | l3_agent_scheduler        | Schedule routers among l3 agents                                                                                                                         |
     | Tag support                                                                                  | tag                       | Enables to set tag on resources.                                                                                                                         |
     | Neutron external network                                                                     | external-net              | Adds external network attribute to network resource.                                                                                                     |
     | Tag support for resources with standard attribute: trunk, policy, security_group, floatingip | standard-attr-tag         | Enables to set tag on resources with standard attribute.                                                                                                 |
     | Neutron Service Flavors                                                                      | flavors                   | Flavor specification for Neutron advanced services                                                                                                       |
     | Network MTU                                                                                  | net-mtu                   | Provides MTU attribute for a network resource.                                                                                                           |
     | Availability Zone                                                                            | availability_zone         | The availability zone extension.                                                                                                                         |
     | Quota management support                                                                     | quotas                    | Expose functions for quotas management per tenant                                                                                                        |
     | If-Match constraints based on revision_number                                                | revision-if-match         | Extension indicating that If-Match based on revision_number is supported.                                                                                |
     | HA Router extension                                                                          | l3-ha                     | Add HA capability to routers.                                                                                                                            |
     | Provider Network                                                                             | provider                  | Expose mapping of virtual networks to physical networks                                                                                                  |
     | Multi Provider Network                                                                       | multi-provider            | Expose mapping of virtual networks to multiple physical networks                                                                                         |
     | Quota details management support                                                             | quota_details             | Expose functions for quotas usage statistics per project                                                                                                 |
     | Address scope                                                                                | address-scope             | Address scopes extension.                                                                                                                                |
     | Neutron Extra Route                                                                          | extraroute                | Extra routes configuration for L3 router                                                                                                                 |
     | Network MTU (writable)                                                                       | net-mtu-writable          | Provides a writable MTU attribute for a network resource.                                                                                                |
     | Subnet service types                                                                         | subnet-service-types      | Provides ability to set the subnet service_types field                                                                                                   |
     | Resource timestamps                                                                          | standard-attr-timestamp   | Adds created_at and updated_at fields to all Neutron resources that have Neutron standard attributes.                                                    |
     | Neutron Service Type Management                                                              | service-type              | API for retrieving service providers for Neutron advanced services                                                                                       |
     | Router Flavor Extension                                                                      | l3-flavors                | Flavor support for routers.                                                                                                                              |
     | Port Security                                                                                | port-security             | Provides port security                                                                                                                                   |
     | Neutron Extra DHCP options                                                                   | extra_dhcp_opt            | Extra options configuration for DHCP. For example PXE boot options to DHCP clients can be specified (e.g. tftp-server, server-ip-address, bootfile-name) |
     | Resource revision numbers                                                                    | standard-attr-revisions   | This extension will display the revision number of neutron resources.                                                                                    |
     | Pagination support                                                                           | pagination                | Extension that indicates that pagination is enabled.                                                                                                     |
     | Sorting support                                                                              | sorting                   | Extension that indicates that sorting is enabled.                                                                                                        |
     | security-group                                                                               | security-group            | The security groups extension.                                                                                                                           |
     | DHCP Agent Scheduler                                                                         | dhcp_agent_scheduler      | Schedule networks among dhcp agents                                                                                                                      |
     | Router Availability Zone                                                                     | router_availability_zone  | Availability zone support for router.                                                                                                                    |
     | RBAC Policies                                                                                | rbac-policies             | Allows creation and modification of policies that control tenant access to resources.                                                                    |
     | Tag support for resources: subnet, subnetpool, port, router                                  | tag-ext                   | Extends tag support to more L2 and L3 resources.                                                                                                         |
     | standard-attr-description                                                                    | standard-attr-description | Extension to add descriptions to standard attributes                                                                                                     |
     | Neutron L3 Router                                                                            | router                    | Router abstraction for basic L3 forwarding between L2 Neutron networks and access to external networks via a NAT gateway.                                |
     | Allowed Address Pairs                                                                        | allowed-address-pairs     | Provides allowed address pairs                                                                                                                           |
     | project_id field enabled                                                                     | project-id                | Extension that indicates that project_id field is enabled.                                                                                               |
     | Distributed Virtual Router                                                                   | dvr                       | Enables configuration of Distributed Virtual Routers.                                                                                                    |
     +----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
     ```

  2. 验证 neutron 代理是否正常

     ```
     # openstack network agent list
     +--------------------------------------+--------------------+--------------+-------------------+-------+-------+---------------------------+
     | ID                                   | Agent Type         | Host         | Availability Zone | Alive | State | Binary                    |
     +--------------------------------------+--------------------+--------------+-------------------+-------+-------+---------------------------+
     | 1deeb676-266d-43e0-b52d-2cacc55735be | DHCP agent         | controller01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
     | 3d12bbd7-a7e4-437b-a54d-7bbb422661f2 | L3 agent           | controller01 | nova              | :-)   | UP    | neutron-l3-agent          |
     | d082bdb5-c470-4e67-b023-c55af9a48a7c | Metadata agent     | controller01 | None              | :-)   | UP    | neutron-metadata-agent    |
     | eb8ad4a8-faa3-4689-aa33-72bde7eeeb0f | Linux bridge agent | controller01 | None              | :-)   | UP    | neutron-linuxbridge-agent |
     +--------------------------------------+--------------------+--------------+-------------------+-------+-------+---------------------------+
     ```

### computer01

- 安装与配置

  1. 组件

     ```
     # yum install openstack-neutron-linuxbridge ebtables ipset -y
     ```

  2. 通用配置

     ```
     # vim /etc/neutron/neutron.conf
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     Keystone

     ```
     [DEFAULT]
     ...
     auth_strategy = keystone
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
     username = neutron
     password = qwe123
     ```

     锁路径

     ```
     [oslo_concurrency]
     ...
     lock_path = /var/lib/neutron/tmp
     ```

  3. 配置 Linuxbridge 代理

     Linuxbridge 代理为实例建立 layer-2 虚拟网络并且处理安全组规则

     ```
     # vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
     ```

     将公共虚拟网络和公共物理网络接口对应起来

     此处 'PROVIDER_INTERFACE_NAME' 即 computer01 节点的 Provider network 网口 (ens32)

     ```
     [linux_bridge]
     physical_interface_mappings = provider:ens32
     ```

     启用 VXLAN overlay 网络

     配置 overlay 网络的物理网络接口的 IP，即 Management network IP (10.0.0.31)

     启用 layer-2 population

     ```
     [vxlan]
     enable_vxlan = true
     local_ip = 10.0.0.31
     l2_population = true
     ```

     安全组及防火墙

     ```
     [securitygroup]
     ...
     enable_security_group = true
     firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
     ```

  4. 计算服务对接 Neutron

     ```
     # vim /etc/nova/nova.conf
     ...
     [neutron]
     ...
     url = http://controller01:9696
     auth_url = http://controller01:35357
     auth_type = password
     project_domain_name = default
     user_domain_name = default
     region_name = RegionOne
     project_name = service
     username = neutron
     password = qwe123
     ```

- 启动服务

     ```
     # systemctl restart openstack-nova-compute.service

     # systemctl enable neutron-linuxbridge-agent.service

     # systemctl start neutron-linuxbridge-agent.service
     ```

- 验证

  1. 验证 neutron 代理是否正常

     ```
     # openstack network agent list
     +--------------------------------------+--------------------+--------------+-------------------+-------+-------+---------------------------+
     | ID                                   | Agent Type         | Host         | Availability Zone | Alive | State | Binary                    |
     +--------------------------------------+--------------------+--------------+-------------------+-------+-------+---------------------------+
     | 1deeb676-266d-43e0-b52d-2cacc55735be | DHCP agent         | controller01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
     | 3d12bbd7-a7e4-437b-a54d-7bbb422661f2 | L3 agent           | controller01 | nova              | :-)   | UP    | neutron-l3-agent          |
     | 5f601fae-ced0-4206-bc2d-402a1934f70f | Linux bridge agent | computer01   | None              | :-)   | UP    | neutron-linuxbridge-agent |
     | d082bdb5-c470-4e67-b023-c55af9a48a7c | Metadata agent     | controller01 | None              | :-)   | UP    | neutron-metadata-agent    |
     | eb8ad4a8-faa3-4689-aa33-72bde7eeeb0f | Linux bridge agent | controller01 | None              | :-)   | UP    | neutron-linuxbridge-agent |
     +--------------------------------------+--------------------+--------------+-------------------+-------+-------+---------------------------+
     ```

## Dashboard

### controller01

- 安装配置

  1. 组件

     ```
     # yum install openstack-dashboard -y
     ```

  2. 配置

     ```
     # vim /etc/openstack-dashboard/local_settings
     ```

     ```
     OPENSTACK_HOST = "controller01"
     ```

     可访问主机

     ```
     ALLOWED_HOSTS = ['controller01', 'computer01']
     ```

     配置 memcached 会话存储服务

     注： 注释掉原有 'CACHES'

     ```
     SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

     CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
            'LOCATION': 'controller01:11211',
        }
     }
     ```

     启用第3版认证 API

     ```
     OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
     ```

     启用 domains 支持

     ```
     OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
     ```

     配置 API 版本

     ```
     OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "volume": 2,
     }
     ```

     通过仪表盘创建用户时的默认域配置为 default

     ```
     OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
     ```

     通过仪表盘创建的用户默认角色配置为 user

     ```
     OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
     ```

  3. 重启web服务器以及会话存储服务

     ```
     # systemctl restart httpd.service memcached.service
     ```

- 验证

  访问 http://controller01/dashboard


## Cinder

块存储服务

### block01

块存储节点提前准备一块独立盘 /dev/sdb，使用 LVM 提供逻辑卷，并通过 iSCSI 协议提供给实例

- 安装配置

  1. 工具包

     ```
     # yum install lvm2 device-mapper-persistent-data -y

     # systemctl enable lvm2-lvmetad.service

     # systemctl start lvm2-lvmetad.service
     ```

  2. LVM 配置

     ```
     # pvcreate /dev/sdb
     Physical volume "/dev/sdb" successfully created.

     # vgcreate cinder-volumes /dev/sdb
     Volume group "cinder-volumes" successfully created
     ```

     LVM 默认会扫描 /dev 下所有设备，改为只扫描 cinder-volume 卷组设备 /dev/sdb

     注：本环境系统盘 /dev/sda 没使用 LVM，不必加入扫描范围

     ```
     # vim /etc/lvm/lvm.conf
     ...
     devices{
     ...
     filter = [ "a/sdb/", "r/.*/"]
     ...
     }
     ```

     测试

     ```
     # vgs -vvvv
     ```

  3. Cinder 组件

     ```
     # yum install openstack-cinder targetcli python-keystone -y
     ```

  4. 配置

     ```
     # vim /etc/cinder/cinder.conf
     ```

     数据库

     ```
     [database]
     ...
     connection = mysql+pymysql://cinder:qwe123@controller01/cinder
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     Keystone

     ```
     [DEFAULT]
     ...
     auth_strategy = keystone
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
     username = cinder
     password = qwe123
     ```

     my_ip

     ```
     [DEFAULT]
     ...
     my_ip = 10.0.0.41
     ```

     LVM

     ```
     volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
     volume_group = cinder-volumes
     iscsi_protocol = iscsi
     iscsi_helper = lioadm
     ```

     启用 LVM 后端

     ```
     [DEFAULT]
     ...
     enabled_backends = lvm
     ```

     Glance

     ```
     [DEFAULT]
     ...
     glance_api_servers = http://controller01:9292
     ```

     锁路径

     ```
     [oslo_concurrency]
     ...
     lock_path = /var/lib/cinder/tmp
     ```

  5. 服务

     先部署控制节点 Cinder

### controller01

- 安装配置

  1. 数据库

     ```
     # mysql -uroot -p

     MariaDB [(none)]> CREATE DATABASE cinder;
     Query OK, 1 row affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'controller01' IDENTIFIED BY 'qwe123';
     Query OK, 0 rows affected (0.00 sec)

     MariaDB [(none)]> quit
     Bye
     ```

  2. cinder 用户相关配置

     ```
     # . admin_openrc
     ```

     创建 cinder 用户

     ```
     # openstack user create --domain default --password-prompt cinder
     User Password:
     Repeat User Password:
     +---------------------+----------------------------------+
     | Field               | Value                            |
     +---------------------+----------------------------------+
     | domain_id           | default                          |
     | enabled             | True                             |
     | id                  | 812d906fb4354b28a06835990eff9d54 |
     | name                | cinder                           |
     | options             | {}                               |
     | password_expires_at | None                             |
     +---------------------+----------------------------------+
     ```

     赋权

     ```
     # openstack role add --project service --user cinder admin
     ```

     创建 cinderv2 和 cinderv3 服务实体

     ```
     # openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | OpenStack Block Storage          |
     | enabled     | True                             |
     | id          | bc0ec7b23e914cca85cffd5039ae095d |
     | name        | cinderv2                         |
     | type        | volumev2                         |
     +-------------+----------------------------------+

     # openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
     +-------------+----------------------------------+
     | Field       | Value                            |
     +-------------+----------------------------------+
     | description | OpenStack Block Storage          |
     | enabled     | True                             |
     | id          | 4db43df009874b34ab0fe0756ca1cdf6 |
     | name        | cinderv3                         |
     | type        | volumev3                         |
     +-------------+----------------------------------+
     ```

     创建 Block Storage service 的 API endpoints

     V2 的

     ```
     # openstack endpoint create --region RegionOne volumev2 public http://controller01:8776/v2/%\(project_id\)s
     +--------------+--------------------------------------------+
     | Field        | Value                                      |
     +--------------+--------------------------------------------+
     | enabled      | True                                       |
     | id           | 6c35ddebf75d42899d031caa0c1f0ac5           |
     | interface    | public                                     |
     | region       | RegionOne                                  |
     | region_id    | RegionOne                                  |
     | service_id   | bc0ec7b23e914cca85cffd5039ae095d           |
     | service_name | cinderv2                                   |
     | service_type | volumev2                                   |
     | url          | http://controller01:8776/v2/%(project_id)s |
     +--------------+--------------------------------------------+

     # openstack endpoint create --region RegionOne volumev2 internal http://controller01:8776/v2/%\(project_id\)s
     +--------------+--------------------------------------------+
     | Field        | Value                                      |
     +--------------+--------------------------------------------+
     | enabled      | True                                       |
     | id           | e4420cc875eb4f708d9e1c332b3fd655           |
     | interface    | internal                                   |
     | region       | RegionOne                                  |
     | region_id    | RegionOne                                  |
     | service_id   | bc0ec7b23e914cca85cffd5039ae095d           |
     | service_name | cinderv2                                   |
     | service_type | volumev2                                   |
     | url          | http://controller01:8776/v2/%(project_id)s |
     +--------------+--------------------------------------------+

     # openstack endpoint create --region RegionOne volumev2 admin http://controller01:8776/v2/%\(project_id\)s
     +--------------+--------------------------------------------+
     | Field        | Value                                      |
     +--------------+--------------------------------------------+
     | enabled      | True                                       |
     | id           | b55ed84172d448daaae8b299fa5ce223           |
     | interface    | admin                                      |
     | region       | RegionOne                                  |
     | region_id    | RegionOne                                  |
     | service_id   | bc0ec7b23e914cca85cffd5039ae095d           |
     | service_name | cinderv2                                   |
     | service_type | volumev2                                   |
     | url          | http://controller01:8776/v2/%(project_id)s |
     +--------------+--------------------------------------------+
     ```

     V3 的

     ```
     # openstack endpoint create --region RegionOne volumev3 public http://controller01:8776/v3/%\(project_id\)s
     +--------------+--------------------------------------------+
     | Field        | Value                                      |
     +--------------+--------------------------------------------+
     | enabled      | True                                       |
     | id           | 307503482bbd48f7946cf1b29cab46b7           |
     | interface    | public                                     |
     | region       | RegionOne                                  |
     | region_id    | RegionOne                                  |
     | service_id   | 4db43df009874b34ab0fe0756ca1cdf6           |
     | service_name | cinderv3                                   |
     | service_type | volumev3                                   |
     | url          | http://controller01:8776/v3/%(project_id)s |
     +--------------+--------------------------------------------+

     # openstack endpoint create --region RegionOne volumev3 internal http://controller01:8776/v3/%\(project_id\)s
     +--------------+--------------------------------------------+
     | Field        | Value                                      |
     +--------------+--------------------------------------------+
     | enabled      | True                                       |
     | id           | fcbbbb2f5bd643798ffe43f063a4da1b           |
     | interface    | internal                                   |
     | region       | RegionOne                                  |
     | region_id    | RegionOne                                  |
     | service_id   | 4db43df009874b34ab0fe0756ca1cdf6           |
     | service_name | cinderv3                                   |
     | service_type | volumev3                                   |
     | url          | http://controller01:8776/v3/%(project_id)s |
     +--------------+--------------------------------------------+

     # openstack endpoint create --region RegionOne volumev3 admin http://controller01:8776/v3/%\(project_id\)s
     +--------------+--------------------------------------------+
     | Field        | Value                                      |
     +--------------+--------------------------------------------+
     | enabled      | True                                       |
     | id           | 765308ee34a04f948b779bd8d78671a5           |
     | interface    | admin                                      |
     | region       | RegionOne                                  |
     | region_id    | RegionOne                                  |
     | service_id   | 4db43df009874b34ab0fe0756ca1cdf6           |
     | service_name | cinderv3                                   |
     | service_type | volumev3                                   |
     | url          | http://controller01:8776/v3/%(project_id)s |
     +--------------+--------------------------------------------+
     ```

  3. 组件

     ```
     # yum install openstack-cinder -y
     ```

  4. 配置

     ```
     # vim /etc/cinder/cinder.conf
     ```

     数据库

     ```
     [database]
     ...
     connection = mysql+pymysql://cinder:qwe123@controller01/cinder
     ```

     RabbitMQ

     ```
     [DEFAULT]
     ...
     transport_url = rabbit://openstack:qwe123@controller01
     ```

     Keystone

     ```
     [DEFAULT]
     ...
     auth_strategy = keystone
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
     username = cinder
     password = qwe123
     ```

     my_ip

     ```
     [DEFAULT]
     ...
     my_ip = 10.0.0.11
     ```

     锁路径

     ```
     [oslo_concurrency]
     ...
     lock_path = /var/lib/cinder/tmp
     ```

     初始化块存储数据库

     ```
     # sh -c "cinder-manage db sync" cinder
     ```

  5. 配置 nova 使用块存储

     ```
     # vim /etc/nova/nova.conf
     ...
     [cinder]
     os_region_name = RegionOne
     ```

  6. 服务

     回到块存储节点 block01 启动 cinder 服务

     ```
     [root@block01 ~]# systemctl enable openstack-cinder-volume.service target.service

     [root@block01 ~]# systemctl restart openstack-cinder-volume.service target.service
     ```

     控制节点继续如下操作

     ```
     # systemctl restart openstack-nova-api.service

     # systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service

     # systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
     ```

- Install and configure the backup service

  https://docs.openstack.org/cinder/pike/install/cinder-backup-install-rdo.html

- 验证

  1. 列出服务组件以验证是否每个进程都成功启动

     ```
     # openstack volume service list
     +------------------+--------------+------+---------+-------+----------------------------+
     | Binary           | Host         | Zone | Status  | State | Updated At                 |
     +------------------+--------------+------+---------+-------+----------------------------+
     | cinder-volume    | block01@lvm  | nova | enabled | up    | 2019-01-29T03:31:29.000000 |
     | cinder-scheduler | controller01 | nova | enabled | up    | 2019-01-29T03:31:29.000000 |
     +------------------+--------------+------+---------+-------+----------------------------+
     ```
