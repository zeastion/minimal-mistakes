服务器两块网卡做 Bond，并配置 Vlan 为 321

## || 关闭 NetworkManager

```bash
# systemctl stop NetworkManager

# systemctl disable NetworkManager
```


## || 修改网卡配置

### 1- bond 配置

```bash
# vim /etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 miimon=100 mode=1
```

### 2- 需要 bond 的两个端口

```bash
# vim /etc/sysconfig/network-scripts/ifcfg-enp130s0f0
TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
NAME=enp130s0f0
UUID=2140300b-1780-4058-9e09-bfe90c34b272
DEVICE=enp130s0f0
ONBOOT=yes
MASTER=bond0
SLAVE=yes

# vim /etc/sysconfig/network-scripts/ifcfg-enp130s0f1
TYPE=Ethernet
BOOTPROTO=dhcp
NAME=enp130s0f1
UUID=a59ad333-13bf-4389-99e2-edc2b3e4c649
DEVICE=enp130s0f1
ONBOOT=yes
MASTER=bond0
SLAVE=yes
```

### 3- 添加 bond0

```bash
# vim /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=static
```

### 4- 配置 Vlan 321 

```bash
# vim /etc/sysconfig/network-scripts/ifcfg-bond0.321
DEVICE=bond0.321
TYPE=Ethernet
ONB00T=yes
BOOTPROTO=none
IPADDR=10.125.144.37
NETMASK=255.255.255.0
GATEWAY=10.125.144.1
DNS1=114.114.114.114
VLAN=yes
```

### 5- 重启

```bash
# reboot
```


## || 测试

分别断开再连接两网口

### 1- 测试 enp130s0f0

```bash
# ifdown enp130s0f0

# tail -f /var/log/messages
Sep 12 15:25:56 F16-7-kubernetes02 kernel: bond0: Removing slave enp130s0f0
Sep 12 15:25:56 F16-7-kubernetes02 kernel: bond0: Releasing active interface enp130s0f0
Sep 12 15:25:56 F16-7-kubernetes02 kernel: bond0: the permanent HWaddr of enp130s0f0 - f4:4c:7f:1d:a7:96 - is still in use by bond0 - set the HWaddr of enp130s0f0 to a different address to avoid conflicts
Sep 12 15:25:56 F16-7-kubernetes02 kernel: bond0: making interface enp130s0f1 the new active one
Sep 12 15:25:56 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.0: removed PHC on enp130s0f0
Sep 12 15:25:57 F16-7-kubernetes02 firewalld: ERROR: UNKNOWN_INTERFACE: 'enp130s0f0' is not in any zone

# ifup enp130s0f0

# tail -f /var/log/messages
Sep 12 15:26:10 F16-7-kubernetes02 kernel: bond0: Adding slave enp130s0f0
Sep 12 15:26:10 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.0: registered PHC device on enp130s0f0
Sep 12 15:26:10 F16-7-kubernetes02 kernel: 8021q: adding VLAN 0 to HW filter on device enp130s0f0
Sep 12 15:26:10 F16-7-kubernetes02 kernel: bond0: Enslaving enp130s0f0 as a backup interface with a down link
Sep 12 15:26:11 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.0 enp130s0f0: detected SFP+: 5
Sep 12 15:26:11 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.0 enp130s0f0: NIC Link is Up 10 Gbps, Flow Control: RX/TX
Sep 12 15:26:11 F16-7-kubernetes02 kernel: bond0: link status definitely up for interface enp130s0f0, 10000 Mbps full duplex
```

### 2- 测试 enp130s0f1

```bash
# ifdown enp130s0f1

# tail -f /var/log/messages
Sep 12 15:30:09 F16-7-kubernetes02 kernel: bond0: Removing slave enp130s0f1
Sep 12 15:30:09 F16-7-kubernetes02 kernel: bond0: Releasing active interface enp130s0f1
Sep 12 15:30:09 F16-7-kubernetes02 kernel: bond0: making interface enp130s0f0 the new active one
Sep 12 15:30:09 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.1: removed PHC on enp130s0f1
Sep 12 15:30:09 F16-7-kubernetes02 firewalld: ERROR: UNKNOWN_INTERFACE: 'enp130s0f1' is not in any zone

# ifup enp130s0f1

# tail -f /var/log/messages
Sep 12 15:31:09 F16-7-kubernetes02 kernel: bond0: Adding slave enp130s0f1
Sep 12 15:31:09 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.1: registered PHC device on enp130s0f1
Sep 12 15:31:09 F16-7-kubernetes02 kernel: 8021q: adding VLAN 0 to HW filter on device enp130s0f1
Sep 12 15:31:09 F16-7-kubernetes02 kernel: bond0: Enslaving enp130s0f1 as a backup interface with a down link
Sep 12 15:31:09 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.1 enp130s0f1: detected SFP+: 6
Sep 12 15:31:09 F16-7-kubernetes02 kernel: ixgbe 0000:82:00.1 enp130s0f1: NIC Link is Up 10 Gbps, Flow Control: RX/TX
Sep 12 15:31:09 F16-7-kubernetes02 kernel: bond0: link status definitely up for interface enp130s0f1, 10000 Mbps full duplex
```