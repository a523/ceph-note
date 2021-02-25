# 启用dashboad

在需要启用的节点上安装

```shell
yum install ceph-mgr-dashboard -y
```

启用

```shell
ceph mgr module enable dashboard
```

设置签名证书

```shell
ceph dashboard create-self-signed-cert
```

创建管理员用户

```shell
ceph dashboard ac-user-create {username} {passwd} administrator
```

查看端口信息

```shell
ceph mgr services | grep dashboard
```

设置固定端口

```shell
ceph config set mgr mgr/dashboard/server_addr $IP
ceph config set mgr mgr/dashboard/server_port $PORT
ceph config set mgr mgr/dashboard/ssl_server_port $PORT
```





