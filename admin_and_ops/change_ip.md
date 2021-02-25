

# 改变集群ip地址

> https://docs.ceph.com/en/latest/rados/operations/add-or-rm-mons
>

集群由单网络改成内外网分离

## 1. 暂停集群

```shell
#ceph osd set noout
#ceph osd set norecover
#ceph osd set norebalance
#ceph osd set nobackfill
#ceph osd set nodown
#ceph osd set pause
```



## 2. 更改 node ip

## 3. 修改 mon ip
导出 mon map
```shell
ceph mon getmap -o {tmp}/{filename}
```
查看map内容
```shell
monmaptool --print {tmp}/{filename}
```
备份map

```shell
cp mon.map old_mon.map
```

删除现有mon









### 一、换机房

用所有监视器的新 IP 地址生成新 monmap ，并注入到集群内的所有监视器。

