# 性能测试结果

## rados bench

**写**

对象大小4M， 线程16

rados bench -p benchmark 60 write --no-cleanup

```shell
Total time run:         60.6268
Total writes made:      1290
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     85.1109
Stddev Bandwidth:       10.6562
Max bandwidth (MB/sec): 108
Min bandwidth (MB/sec): 44
Average IOPS:           21
Stddev IOPS:            2.66405
Max IOPS:               27
Min IOPS:               11
Average Latency(s):     0.751312
Stddev Latency(s):      0.322423
Max latency(s):         2.17188
Min latency(s):         0.24994
```

**读**(随机)

rados bench -p benchmark 60 rand

```shell
Total time run:       60.5095
Total reads made:     2379
Read size:            4194304
Object size:          4194304
Bandwidth (MB/sec):   157.264
Average IOPS:         39
Stddev IOPS:          3.61303
Max IOPS:             48
Min IOPS:             32
Average Latency(s):   0.404598
Max latency(s):       2.33844
Min latency(s):       0.00449078
```

## 优化

### os优化

```shell
# 最大文件数
cat /proc/meminfo | grep MemTotal | awk '{print $2}' > /proc/sys/fs/file-max
# 预读
/sbin/blockdev --setra 8192 /dev/sdb  # 修改所有数据盘
# 默认的请求队列数
echo 512 > /sys/block/sdb/queue/nr_requests # 修改所有数据盘
```

### ceph配置

```shell
[global]
max_open_files = 131072
[osd]
osd_max_write_size = 512
osd_op_threads = 8
osd_disk_threads = 4
objecter_inflight_ops = 819200
osd_memory_target = 4294967296
```



## cosbench

64 kb   线程12

| Op-Type | Op-Count   | Byte-Count | Avg-ResTime | Avg-ProcTime | Throughput  | Bandwidth  | Succ-Ratio |
| :------ | ---------- | ---------- | ----------- | ------------ | ----------- | ---------- | ---------- |
| read    | 51.86 kops | 3.32 GB    | 65.78 ms    | 33.92 ms     | 172.87 op/s | 11.06 MB/S | 94.81%     |
| write   | 54.08 kops | 3.46 GB    | 66.44 ms    | 65.71 ms     | 180.28 op/s | 11.54 MB/S | 100%       |



## 存储升级后的性能测试结果

万兆网络，内外网分离， osd 数据盘（机械盘）24 个， ssd （缓存和索引）4个

测试工具：obscmdbench

|                      |           4k \ 320并发            |           64K \ 320并发           |            4M \ 128并发            |         100M \ 32并发         |
| :------------------: | :-------------------------------: | :-------------------------------: | :--------------------------------: | :---------------------------: |
| 写 (带宽\|TPS\|时延) | 38.11 MB/s \| 9756.95 \| 32.48ms  | 380.93 MB/s \| 6094.87 \| 52.17ms | 764.06 MB/s \| 195.60 \| 652.65ms  | 781.68 MB/s \| 7.82 \| 4.03s  |
| 读(带宽\|TPS\|时延)  | 27.66 MB/s \| 7081.72  \| 44.93ms | 350.52 MB/s \| 5608.29 \| 56.77ms | 780.53 MB/s \| 199.81 \| 638.83 ms | 811.23 MB/s  \| 8.11 \| 3.87s |
|                      |                                   |                                   |                                    |                               |



