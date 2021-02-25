## 删除 osd 

删除 CRUSH 图的对应 OSD 条目

```
ceph osd crush remove {name}
# 如
ceph osd crush remove osd.1
```

删除 OSD 认证密钥

```shell
ceph auth del osd.1
```

删除 OSD

```shell
ceph osd rm {osd-num}
#for example
ceph osd rm 1
```

编辑 ceph.conf， 删掉对应项

