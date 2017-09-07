RBD & Filesystem & Object Storage

## || 块设备 RBD 

Ceph Block Device, the block storage component of Ceph

- ceph-client 节点

### 0- ceph-client 准备工作

```bash
# vim /etc/hosts
10.50.50.137    ceph-admin
10.50.50.138    ceph-client
10.50.50.134    ceph01
10.50.50.135    ceph02
10.50.50.136    ceph03

# apt-get install ntp -y

# useradd -d /home/zeastion -m zeastion
# passwd zeastion

# echo "zeastion ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/zeastion
# chmod 0440 /etc/sudoers.d/zeastion
```

- ceph-admin 节点

### 1- ceph-admin 添加 ceph-client 域名

```bash
# vim /etc/hosts
10.50.50.137    ceph-admin
10.50.50.138    ceph-client
10.50.50.134    ceph01
10.50.50.135    ceph02
10.50.50.136    ceph03
```

### 2- 免密登录

```bash
# ssh-copy-id zeastion@ceph-client
```

### 3- 给 ceph-client 安装 ceph

```bash
# ceph-deploy install ceph-client
```

### 4- 把配置文件和 keyring 传给 ceph-client

```bash
# ceph-deploy admin ceph-client
```
- ceph-client 节点

### 5- 在 ceph-client 上给 keyring 加读权限

```bash
# chmod +r /etc/ceph/ceph.client.admin.keyring
```

### 6- 创建一个块设备 image

```bash
# rbd create foo --size 4096
```

查看

```bash
# rbd info foo
rbd image 'foo':
        size 4096 MB in 1024 objects
        order 22 (4096 kB objects)
        block_name_prefix: rbd_data.10ce238e1f29
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        flags: 
```

### 7- 把 image 映射为块设备

```bash
# sudo rbd map foo --name client.admin
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail" or so.
rbd: map failed: (6) No such device or address
```

kernel 对 image 的部分特性不支持，将其 disable 掉

```bash
# rbd feature disable foo exclusive-lock object-map fast-diff deep-flatten

# rbd info foo
rbd image 'foo':
        size 4096 MB in 1024 objects
        order 22 (4096 kB objects)
        block_name_prefix: rbd_data.10ce238e1f29
        format: 2
        features: layering
        flags: 
```

再次映射

```bash
# sudo rbd map foo --name client.admin
/dev/rbd0
```

### 8- 创建文件系统

```bash
# sudo mkfs.ext4 -m0 /dev/rbd/rbd/foo
mke2fs 1.42.9 (4-Feb-2014)
Discarding device blocks: done                            
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=1024 blocks, Stripe width=1024 blocks
262144 inodes, 1048576 blocks
0 blocks (0.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=1073741824
32 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done 
```

### 9- 挂载此文件系统

```bash
# sudo mkdir /mnt/ceph-block-device

# sudo mount /dev/rbd/rbd/foo /mnt/ceph-block-device/

# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            981M  4.0K  981M   1% /dev
tmpfs           199M  920K  198M   1% /run
/dev/sda1        27G  1.9G   25G   8% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
none            5.0M     0  5.0M   0% /run/lock
none            992M     0  992M   0% /run/shm
none            100M     0  100M   0% /run/user
/dev/rbd0       3.9G  8.0M  3.8G   1% /mnt/ceph-block-device
```


## || 文件系统 Filesystem

### 0- 确保有元数据服务器

略

- ceph-client 节点

### 1- 创建文件系统

```bash
# ceph osd pool create cephfs_data 50
pool 'cephfs_data' created

# ceph osd pool create cephfs_metadata 50
pool 'cephfs_metadata' created

# ceph fs new myfs cephfs_metadata cephfs_data
new fs with metadata pool 10 and data pool 9
```

### 2- 查看元数据服务器信息（fsmap）

```bash
# ceph -s
    cluster 5651d8c0-317f-404f-b22f-21f5496f1b07
     health HEALTH_OK
     monmap e3: 3 mons at {ceph01=10.50.50.134:6789/0,ceph02=10.50.50.135:6789/0,ceph03=10.50.50.136:6789/0}
            election epoch 10, quorum 0,1,2 ceph01,ceph02,ceph03
      fsmap e5: 1/1/1 up {0=ceph01=up:active}
     osdmap e43: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v7200: 232 pgs, 10 pools, 134 MB data, 237 objects
            21717 MB used, 58349 MB / 80066 MB avail
                 232 active+clean

# ceph mds stat
e5: 1/1/1 up {0=ceph01=up:active}
```

### 3- 创建密钥文件

```bash
# cat /etc/ceph/ceph.client.admin.keyring 
[client.admin]
        key = AQBOa6dZlGydIhAAXTBPCIVAoKs1q0F2d4VpqA==
        caps mds = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"

# echo "AQBOa6dZlGydIhAAXTBPCIVAoKs1q0F2d4VpqA==" > admin.secret

# chmod 600 admin.secret
```

### 4- 挂载

-a- 把 Ceph FS 挂载为内核驱动

指向 MDS 地址 6789 端口

```bash
# sudo mount -t ceph 10.50.50.134:6789:/ /mnt/mycephfs -o name=admin,secretfile=admin.secret
mount: wrong fs type, bad option, bad superblock on 10.50.50.134:6789:/,
       missing codepage or helper program, or other error
       (for several filesystems (e.g. nfs, cifs) you might
       need a /sbin/mount.<type> helper program)
       In some cases useful info is found in syslog - try
       dmesg | tail  or so

# dmesg |tail
...
[234235.078224] libceph: bad option at 'secretfile=admin.secret'
```

需要安装 ceph-fs-common 包

```bash
# sudo apt-get install ceph-fs-common

# sudo mount -t ceph 10.50.50.134:6789:/ /mnt/mycephfs -o name=admin,secretfile=admin.secret

# df -h
Filesystem           Size  Used Avail Use% Mounted on
...
10.50.50.134:6789:/   79G   22G   57G  28% /mnt/mycephfs
```

-b- 把 Ceph FS 挂载为用户空间文件系统（FUSE）

```bash
# sudo apt-get install ceph-fuse

# sudo mkdir /mnt/mycephfuse

# sudo ceph-fuse -k /etc/ceph/ceph.client.admin.keyring -m 10.50.50.134:6789 /mnt/mycephfuse/
ceph-fuse[10074]: starting ceph client
2017-09-04 10:37:35.903181 7fe27f234ec0 -1 init, newargv = 0x55ba8ca7c660 newargc=11
ceph-fuse[10074]: starting fuse

$ df -h
...
10.50.50.134:6789:/   79G   22G   57G  28% /mnt/mycephfs
ceph-fuse             79G   22G   57G  28% /mnt/mycephfuse
```


## || 对象存储 Object Storage

- ceph-admin 节点

### 1- 安装 Ceph 对象网管

```bash
# ceph-deploy install --rgw ceph-client
```

### 2- 新建 Ceph 对象网关实例

```bash
# ceph-deploy rgw create ceph-client
```

- ceph-client 节点 

### 3- 配置对象网关实例

在 [global] 后面增加 [client.rgw.<client-node>] 小节，<client-node> 即 hostname -s 输出的节点短名称

```bash
# sudo vim /etc/ceph/ceph.conf
...
[client.rgw.ceph-client]
rgw_frontends = "civetweb port=8080"
```

重启服务

```bash
# sudo netstat -ntlp | grep rados
tcp        0      0 0.0.0.0:7480            0.0.0.0:*               LISTEN      11127/radosgw

# sudo service radosgw restart id=rgw.ceph-client
radosgw stop/waiting
radosgw (ceph/rgw.ceph-client) start/running, process 11489

# sudo netstat -ntlp | grep rados                  
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      11489/radosgw
```

访问

![rgw-web](http://ov30w4cpi.bkt.clouddn.com/rgw8080.png)