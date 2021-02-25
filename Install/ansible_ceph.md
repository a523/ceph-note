# 用 ansible 安装 ceph

## 说明

ansible 是声明式的自动化运维工具， 你只要声明（play-book）你需要达到一个什么样的状态，运行play-book之后会去达到这个状态， 如果已经是这个状态， 不执行操作。

## 部署节点安装ansible

```shell
yum install -y epel-release ansible python-netaddr
```

## 下载ceph-ansible

```shell
git clone -b stable-4.0 https://github.com/ceph/ceph-ansible.git --recursive
cd ceph-ansible
python3 -m venv venv
source  venv/bin/activate
pip install -r requirements.txt
```

## 定义需要控制的hosts

vim  /etc/ansible/hosts

添加如下内容，这里仅以部署对象存储为例，

如果你需要部署文件存储， 你还需要指定mdss节点

```shell
# 这个文件表示你需要控制的节点
# 节点按角色分类，【】内就是角色名
[mons]  
ceph2
ceph3
ceph4

[osds]
ceph2
ceph3
ceph4

[rgws]
ceph4

[clients]
ceph4

#[grafana-server]
#ceph4
```

## 查看块设备

```shell
lsblk
```

## 修改ceph集群默认配置

```shell
cd group_vars/
# 这个文件夹里面主要定义了你部署ceph时的一些参数
```

### 通过什么方式部署ceph 、外网和内网等信息

all.yml -- 所有节点都会影响的配置文件

> *必须设置的项目*
>
>- `ceph_origin`
>- `ceph_stable_release`
>- `public_network`
>- `monitor_interface` or `monitor_address`

```yaml
configure_firewall: True # ansible 帮你配置跟ceph相关的防火墙
ceph_origin: repository
ceph_repository: community
ceph_mirror: http://download.ceph.com
ceph_stable_release: nautilus
ceph_stable_repo: "{{ ceph_mirror }}/rpm-{{ ceph_stable_release }}"
ceph_stable_redhat_distro: el7
monitor_interface: enp133s0
journal_size: 5120
public_network: 172.19.106.0/0
cluster_network: 172.19.106.0/0

osd_objectstore: bluestore

## Rados Gateway options
radosgw_frontend_type: beast
radosgw_frontend_port: 12345
radosgw_interface: "{{monitor_interface}}"
radosgw_num_instances: 3

dashboard_admin_password: password
grafana_admin_password: admin
```

osds.yml  --影响osd的配置文件

```yaml
devices:  # 数据盘
  - /dev/sdc
  - /dev/sdd
dedicated_devices:  # db盘
  - /dev/sdb、
```

其他可能有用的

- Osd_auto_discovery: True # 自动发现osd


### 集群本身的预配置

Ceph.conf

修改 all.yml 中的 `ceph_conf_overrides` 选项

```yaml
ceph_conf_overrides:
  global:
    osd_pool_default_pg_num: 16
    osd_pool_default_pgp_num: 16
    osd_pool_default_size: 3
  mon:
    mon_all_pool_create: true
```



## 部署对象存储

rgws.yml  --网关节点相关的配置文件

> *必须设置的*
>
> - `radosgw_interface or radosgw_address`

```yaml
rgw_create_pools:
  defaults.rgw.buckets.data:
    pg_num: 8

  defaults.rgw.buckets.index:
    pg_num: 8

###########
# SYSTEMD #
###########
# ceph_rgw_systemd_overrides will override the systemd settings
# for the ceph-rgw services.
# For example,to set "PrivateDevices=false" you can specify:
ceph_rgw_systemd_overrides:
  Service:
    PrivateDevices: False
```

## 开始部署

```shell
cd ..
ansible-playbook site.yml
```

最后显示结果failed = 0 即为成功。

## 管理用户添加到ceph组

```shell
sudo usermod -a -G ceph cadmin
```

