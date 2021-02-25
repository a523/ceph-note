# 安装操作系统及预设置

## 硬件配置说明

两个 480G 的STAT ssd 组成 RADI1 用来安装系统

一个 NVME 2T 的 ssd 用作加速盘，再后续ceph配置中会用到。

其他机械硬盘是数据盘。

## 版本

centos 7

最小化安装， 选择开发者工具和debug tools

时区：上海， 语言：英文

增加一个管理员用户cadmin （后续用该用户部署ceph， 用户名随意， 这里用cadmin代替）

主机名ceph1...n

建议在此步骤同时设置固定ip

也可参考下个步骤，在系统安装完成之后设置固定ip

## 设置固定ip

```shell
[cadmin@ceph3 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-em1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
UUID=82ebcb8f-3230-4489-8ed3-79dfb42333fe
DEVICE=em1
ONBOOT=yes
IPADDR=192.168.8.52  # 根据实际情况修改
PREFIX=24
GATEWAY=192.168.8.254  # 根据实际情况修改
DNS1=114.114.114.114
IPV6_PRIVACY=no
```

## 升级系统

```shell
yum update -y
reboot
```

## 升级内核
省略……

## 常用软件安装

````shell
yum install net-tools vim wget yum-plugin-priorities epel-release -y 
````

## sudo 免密码

为了后面自动化部署，必须配置

```shell
# 创建部署 ceph 专用用户
sudo useradd -d /home/{username} -m {username}
sudo passwd {username}

echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
sudo chmod 0440 /etc/sudoers.d/{username}
	
sudo usermod -a -G wheel {username} 
```

## 同步HOSTS

往**所有节点**/etc/hosts，增加如下内容

```shell
192.168.8.50	ceph1
192.168.8.51	ceph2
192.168.8.52	ceph3
```

## 配置部署节点到所有节点的免密登录

```shell
# 在部署节点执行
# 此步骤不要使用sudo 或root 帐号
ssh-keygen
ssh-copy-id {username}@ceph1
ssh-copy-id {username}@ceph2
ssh-copy-id {username}@ceph3
```

## 配置NTP

ceph对时间敏感，要求各节点时间保持一致

```shell
sudo yum install ntp ntpdate  -y
```

（可选）修改/etc/ntp.conf

```shell
server 0.cn.pool.ntp.org iburst
server 1.cn.pool.ntp.org iburst
server 2.cn.pool.ntp.org iburst
server 3.cn.pool.ntp.org iburst
```

```shell
# 配置NTP
sudo ntpdate pool.ntp.org 
sudo systemctl restart ntpdate
sudo systemctl restart ntpd
sudo systemctl enable ntpd
sudo systemctl enable ntpdate
```

写入硬件时间, 避免重启失效

```shell
hwclock -w
```

参考：[Network Time Protocol daemon (简体中文)](https://wiki.archlinux.org/index.php/Network_Time_Protocol_daemon_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#NTP_%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%A8%A1%E5%BC%8F)

## 关闭SELinux

```shell
# 修改 /etc/selinux/config
SELINUX=disabled 
```

