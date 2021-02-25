# 手动部署ceph

## 添加源

```shell
[xin@centos5 ~]$ cat /etc/yum.repos.d/ceph.repo 
[ceph_stable]
baseurl = http://mirrors.ustc.edu.cn/ceph/rpm-nautilus/el7/$basearch
gpgcheck = 1
gpgkey = https://download.ceph.com/keys/release.asc
name = Ceph Stable $basearch repo
priority = 2

[ceph_stable_noarch]
baseurl = http://mirrors.ustc.edu.cn/ceph/rpm-nautilus/el7/noarch
gpgcheck = 1
gpgkey = https://download.ceph.com/keys/release.asc
name = Ceph Stable noarch repo
priority = 2
```

## 安装

```shell
# 安装依赖
sudo yum install snappy leveldb gdisk python-argparse gperftools-libs
# 安装ceph
sudo yum install ceph -y
```

## 添加配置文件（可选）

```shell
[xin@centos5 ~]$ cat /etc/ceph/ceph.conf 
fsid = 9d457fc8-6ba0-4ba2-8416-ff66722a816a  # 通过uuidgen得到
mon initial members = centos5  # 初始mon
mon host = 192.168.22.5
```

## 配置秘钥

```shell
sudo ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'

sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring

sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring

sudo chown ceph:ceph /tmp/ceph.mon.keyring

```

## 生成MON map

