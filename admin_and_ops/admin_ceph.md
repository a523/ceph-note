# ceph 常见管理命令

## 集群容量

```shell
sudo ceph df
```

## 查看所有的池

```shell
sudo rados lspools
```

## 列出某个池的所有对象

```shell
sudo rados -p default.rgw.buckets.data ls
```

## 追踪一个对象存在哪里

```shell
ceph osd map <poolname> <objectname> {<nspace>} : find pg for <object> in <pool> with [namespace]
```

例如：

```shell
[xin@centos4 ~]$ sudo ceph osd map default.rgw.buckets.data install_os.md
osdmap e52 pool 'default.rgw.buckets.data' (1) object 'install_os.md' -> pg 1.daab108c (1.c) -> up ([3,4,2], p3) acting ([3,4,2], p3)

# 对象install_os.md在pg 1.c 中， 在osd[3, 4, 2]有副本
```

## crash map

查看crash

```shell
ceph osd crush tree
```



## 启动单个服务

```shell
systemctl start ceph-osd@1 # 启动该节点上id为1的osd服务
systemctl start ceph-osd.target # 启动该节点上所有的osd
```

## 查看每个osd 的容量和pg

```shell
ceph osd df
```

## 查看运行时配置

需在对应节点执行

```shell
ceph daemon osd.0 config show | less # 比如这里是查看osd.0的配置， 则需要再osd.0所在的那个节点执行
```

MON的角色

> 1. **Leader**: Leader 是实现最新 Paxos 版本的第一个监视器。
> 2. **Provider**: Provider 有最新集群运行图的监视器，但不是第一个实现最新版。
> 3. **Requester:** Requester 落后于 leader ，重回法定人数前，必须同步以获取关于集群的最新信息。

有了这些角色区分， leader就 可以给 provider 委派同步任务，这会避免同步请求压垮 leader 、影响性能。在下面的图示中， requester 已经知道它落后于其它监视器，然后向 leader 请求同步， leader 让它去和 provider 同步。

## 查看指定PG信息

```shell
# 查询pg在哪些osd 上
ceph pg map {pg-num}

```

查询pg 为什么被标记为down

```shell
# 列出所有pg
ceph pg ls
ceph pg 0.5 query 
# recovery_state 字段
```

### 对象找不到

#### 现象：

unfound

```shell
ceph health detail
HEALTH_WARN 1 pgs degraded; 78/3778 unfound (2.065%)
pg 2.4 is active+degraded, 78 unfound
```

#### 原因：

一个 PG 的数据存储在 ceph-osd 1 和 2 上：

- 1 挂了；
- 2 独自处理一些写动作；
- 1 起来了；
- 1 和 2 重新互联， 1 上面丢失的对象加入队列准备恢复；
- 新对象还未拷贝完， 2 挂了。

首先， 确认哪些对象找不到了

## 设置集群暂停自动均衡数据
```shell
ceph osd set 	noout
# 解除 
ceph osd unset noout
```

## 更换硬盘

参考：https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html/operations_guide/handling-a-disk-failure#replacing-an-osd-drive-while-retaining-the-osd-id-ops

容易想到的是可以通过删除osd 再添加osd 的方法更换硬盘， 但是原来的osd id 会变化。

下面参考的方法可以保证osd id 和CRUSH map 的不变

1. 确保可以安全地销毁osd

   ```shell
   while ! sudo ceph osd safe-to-destroy osd.{id}; do sleep 10 ; done
   ```

2. 销毁osd

   ```shell
   ceph osd destroy {id} --yes-i-really-mean-it
   ```

3. 去掉新的硬盘的数据

   ```shell
   ceph-volume lvm zap /dev/sdX
   ```

4. 创建osd 

   ```shell
   ceph-volume lvm create --osd-id {id}  --data /dev/sdX
   ```

## 删除OSD

1. 踢出出集群

   ```shell
   ceph osd out {osd-num}
   ```

   关注数据迁移

   ```shell
   ceph -w
   ```

2. 停止OSD进程

   ```shel
   ssh {osd-host}
   sudo systemctl stop ceph-osd@{osd-num}
   ```

3. 移除osd从cluster map

   ```shell
   ceph osd purge {id} --yes-i-really-mean-it
   ```

4. 修改ceph.conf中相关osd 的配置，并同步

5. 销毁对于分区

   ```shell
   sudo ceph-volume lvm zap /dev/{drive} --destroy
   ```

## 调整 pg 均衡

调整pg均衡关系到集群的性能，同时调整pg的过程对集群会产生很大的波动，建议在集群刚搭建好的时候调整一次，在线上的集群，应避免大幅调整。

```shell
ceph  balancer status
ceph balancer on
```

调整方法参考：https://zphj1987.com/2020/06/17/ceph-balancer/

## 常见告警

1. ### pgs inconsistent 
    pg 不一致， 
    找出具体不一致的pg

    ```shell
    ceph health detail
    ```
    修复pg
    ```shell
    ceph pg repair {pg.id}
    ```
    参考：https://docs.ceph.com/docs/master/rados/operations/pg-repair/
  
2. ### stale
    某pg 中的主osd 挂了（down），有一段时间没上mon 报告
    
    找出卡主的pg
    
    ```shell
    ceph pg dump_stuck [unclean|inactive|stale|undersized|degraded]
    ```

## 暂停集群
https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/administration_guide/override-ceph-behavior

```shel
ceph osd set pausel  # 暂停读写
```