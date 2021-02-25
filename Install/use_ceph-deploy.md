# 通过ceph-deploy 部署

## 安装依赖源

```shel
sudo yum install -y epel-release 
```

## 添加ceph源

```shell
[xin@centos5 ~]$ cat /etc/yum.repos.d/ceph.repo 
[ceph]
baseurl = http://mirrors.ustc.edu.cn/ceph/rpm-nautilus/el7/$basearch
gpgcheck = 1
gpgkey = https://download.ceph.com/keys/release.asc
name = Ceph Stable $basearch repo
priority = 2

[ceph_noarch]
baseurl = http://mirrors.ustc.edu.cn/ceph/rpm-nautilus/el7/noarch
gpgcheck = 1
gpgkey = https://download.ceph.com/keys/release.asc
name = Ceph Stable noarch repo
priority = 2
```

## 安装ceph

```shell
# 管理节点
sudo yum install ceph-deploy -y
# 所有需要部署的ceph节点
sudo yum install ceph -y
# 你也可以使用
ceph-deploy install --no-adjust-repos  {nodes} # --no-adjust-repos选项不会修改你已经配置好的repo
```

## 配置管理节点ssh config （可选）

修改 ~/.ssh/config

```
Host ceph-server
        Hostname ceph-server.fqdn-or-ip-address.com
        User cadmin
```

```shell
sudo chmod 600 ~/.ssh/config
```

## ssh免密登录，同步ssh公钥

把管理节点的公钥同步到所有要部署的节点， 如果管理节点也是集群的一个节点则包括本身也要复制

```shell
ssh-keygen
ssh-copy-id {nodes}
```

## 防火墙

```shell
sudo firewall-cmd --zone=public --add-port 80/tcp --permanent  # 网关端口
sudo firewall-cmd --get-services | grep ceph
sudo firewall-cmd --permanent --zone=public --add-service=ceph
sudo firewall-cmd --permanent --zone=public --add-service=ceph-mon
sudo firewall-cmd --reload
sudo firewall-cmd --list-all 
```



## 部署

创建一个管理目录

```shell
mkdir my-cluster
cd my-cluster
```

开始部署一个新的集群， 创建一个配置文件ceph.conf还有秘钥(执行ceph-deploy请不要用sudo，下同)

```shell
ceph-deploy new {initial-mon-node1} {initial-mon-node2} {initial-mon-node3}
# 可以指定--public-network 等选项
```

## 部署初始mon	

```shell
ceph-deploy mon create-initial
```

## 允许当前用户读秘钥权限

```shell
sudo chmod +r /etc/ceph/ceph.client.admin.keyring 
```

## 拷贝管理秘钥及配置文件到节点默认目录

```shell
ceph-deploy admin {nodes}
```

## 部署MGR

```shell
ceph-deploy mgr create {nodes}
```

## 添加OSD

如果需要db区和数据区分开部署，且多个osd共享一块NVME，需要先给NVME分区，ceph-deploy不会自动分区

### 分区NVME

可以用分区工具fdisk 或者 parted

**NOTE:**  鉴于我们集群中的硬件配置并不一致，共有两种机型，采取如下策略。

>  在 12 盘位的机器上， 把 nvme 分成 12 等分， 用作“缓存”，和各个机械盘组成 osd。即单个机器上总共 12 个 osd， 每个 osd 等于一个机械盘12T+150G NVME 


请使用 **GPT** 分区

#### fdisk

```shell
sudo fdisk /dev/nvme0n1
g
n
+150G
n
...
```

#### parted

```shell
parted /dev/nvme0n1 
mklabel gpt
mkpart primary 1 25%  # 25% 分区结束值，根据情况调整，
```

**利用 fdisk 批量自动分区**

新建fq.txt 文件，输入如下内容

```shell
g
n


+150G
n


+150G
n


+150G
n


+150G
n


+150G
n


+150G
n



w
```

```shell
fdisk /dev/nvme0n1 < fq.txt
```

*有关分区的更详细步骤请参考文后参考资料*

### 创建OSD

```shell
ceph-deploy osd create node1 --data /dev/sdb --block-db /dev/nvme0n1p1
```

**NOTE：**

有些硬盘可能是旧硬盘，上面还有分区信息，创建 osd 的时候会出错，通过如下命名删除

```shell
ceph-volume lvm zap /dev/sdb  --destroy
```

## 批量创建osd（osd 数量较多时可选）

如果osd比较多， 你可以写个脚本批量创建osd，参考脚本如下

Create_osd.sh  (运行该脚本同样不要用sudo)

```shell
#!/bin/bash
for node in ceph1 ceph2 ceph3
do
j=1
for i in {a..f}
do
ceph-deploy osd create ${node} --data /dev/sd${i} --block-db /dev/nvme0n1p${j} 
((j=${j}+1))
done

ceph-deploy osd create ${node} --data /dev/nvme0n1p7  # 额外用作元数据池的osd
done
```



## 部署RGW

在需要安装RGW 的节点执行

```shell
sudo yum install ceph-radosgw -y
```

### 在多节点上RGW实例

#### 修改配置文件

```shell
[xin@centos6 my-cluster]$ cat ~/my-cluster/ceph.conf 
……

[client.rgw.centos6] 
host = centos6
keyring = /var/lib/ceph/radosgw/ceph-rgw.centos6/keyring 
log file = /var/log/ceph/ceph-rgw-centos6.log 
rgw frontends = beast endpoint=192.168.22.106:8080 
rgw thread pool size = 512

[client.rgw.centos7] 
host = centos7
keyring = /var/lib/ceph/radosgw/ceph-rgw.centos7/keyring 
log file = /var/log/ceph/ceph-rgw-centos7.log 
rgw frontends = beast endpoint=192.168.22.106:8081
rgw thread pool size = 512
```

#### 同步配置文件

```shell
ceph-deploy config push {nodes}
```

#### 启动服务

```shell
ceph-deploy rgw create {node}:{rgw_name}
```

## 预先创建data池（可选）

data池和index 池在用户上传数据会自动创建， 你也可以手动创建

```shell
# 创建池
ceph osd pool create default.rgw.buckets.data 128 128

ceph osd pool create default.rgw.buckets.index 256 256

# 池为rgw使用
ceph osd pool application enable default.rgw.buckets.data rgw

ceph osd pool application enable default.rgw.buckets.index rgw
```

有关pg的计算，请参考[该网站](https://ceph.com/pgcalc/)

你还可以定制CRUSH RULE

### 如何通过ceph-deploy 管理一个已经存在的集群

```shell
ceph-deploy config pull HOST # 拉去配置
ceph-deploy gatherkeys MONHOST  # 收集秘钥
```



## 参考资料

ceph-deploy

https://docs.ceph.com/ceph-deploy/docs/

分区

https://www.xncoding.com/2017/03/14/linux/disk-partition.html

[https://zphj1987.com/2016/06/24/parted%E5%88%86%E5%8C%BA%E5%AF%B9%E9%BD%90/](https://zphj1987.com/2016/06/24/parted分区对齐/)

https://docs.ceph.com/docs/mimic/ceph-volume/lvm/prepare/

