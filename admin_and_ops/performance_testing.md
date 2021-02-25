# 性能测试

## 优化

### ceph.conf

```shell
[global]
max_open_files = 131072
rgw_num_rados_handles = 64 
rgw_max_chunk_size = 4194304
rgw_override_bucket_index_max_shards = 64

[osd]
osd_max_write_size = 512
osd_op_threads = 8
osd_disk_threads = 4
objecter_inflight_ops = 819200
osd_memory_target = 8589934592
osd_op_log_threshold  = 50
osd_max_write_size = 90
```

## 集群内部测试（rados bench）

**写**

对象大小4M， 线程16

```shell
rados bench -p benchmark 60 write --no-cleanup
```

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

```shell
rados bench -p benchmark 60 rand
```

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

---

## 外部访问测试(cosbench)

### 安装cosbench

```shell
wget https://github.com/intel-cloud/cosbench/releases/download/v0.4.2.c4/0.4.2.c4.zip
# 这里下载的并非最新版，最新版有bug
yum install java-1.7.0-openjdk nmap-ncat
unzip 0.4.2.c4.zip
cd 0.4.2.c4
./start-all.sh
```

运行成功输出

```shell
Successfully started cosbench driver!
Listening on port 0.0.0.0/0.0.0.0:18089 ... 
Persistence bundle starting...
Persistence bundle started.
----------------------------------------------
!!! Service will listen on web port: 18088 !!!
----------------------------------------------

======================================================

Launching osgi framwork ... 
Successfully launched osgi framework!
Booting cosbench controller ... 
.
Starting    cosbench-log_0.4.2    [OK]
Starting    cosbench-tomcat_0.4.2    [OK]
Starting    cosbench-config_0.4.2    [OK]
Starting    cosbench-core_0.4.2    [OK]
Starting    cosbench-core-web_0.4.2    [OK]
Starting    cosbench-controller_0.4.2    [OK]
Starting    cosbench-controller-web_0.4.2    [OK]
Successfully started cosbench controller!
Listening on port 0.0.0.0/0.0.0.0:19089 ... 
Persistence bundle starting...
Persistence bundle started.
----------------------------------------------
!!! Service will listen on web port: 19088 !!!
----------------------------------------------

```

访问web页面`http://{ip}:19088/controller/index.html` 

### 配置 workloads

**COSBench** 可以web 编辑workloads 也可以通过上传xml格式的配置文件识别workloads。

通过web页面配置workloads目前没看到支持s3协议， 所以得在本地手动文本编辑workloads，然后上传至web。

下载s3 workloads 的模板

```shell
wget https://raw.githubusercontent.com/intel-cloud/cosbench/master/release/conf/s3-config-sample.xml
```

用编辑器打开并按自己配置修改里面的内容

```shell
code s3-config-sample.xml
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<workload name="s3-sample" description="sample benchmark for s3">

  <storage type="s3" config="accesskey=<accesskey>;secretkey=<scretkey>;proxyhost=<proxyhost>;proxyport=<proxyport>;endpoint=<endpoint>" />

  <workflow>
    <!-- 创建桶 -->
    <workstage name="init">
      <work type="init" workers="1" config="cprefix=s3testqwer;containers=r(1,2)" />
    </workstage>
    
    <!-- 预写少部分数据，用于后面的读 -->
    <workstage name="prepare">
      <work type="prepare" workers="1" config="cprefix=s3testqwer;containers=r(1,2);objects=r(1,10);sizes=c(64)KB" />
    </workstage>
    
		<!--  正式测试部分 -->
    <workstage name="main">
      <work name="main" workers="8" runtime="30">
        <operation type="read" ratio="100" config="cprefix=s3testqwer;containers=u(1,2);objects=u(1,10)" />
        <operation type="write" ratio="20" config="cprefix=s3testqwer;containers=u(1,2);objects=u(11,20);sizes=c(64)KB" />
      </work>
    </workstage>

    <!-- 清除 -->
    <workstage name="cleanup">
      <work type="cleanup" workers="1" config="cprefix=s3testqwer;containers=r(1,2);objects=r(1,20)" />
    </workstage>

    <!-- 删除桶 -->
    <workstage name="dispose">
      <work type="dispose" workers="1" config="cprefix=s3testqwer;containers=r(1,2)" />
    </workstage>

  </workflow>

</workload>
```

下面是一个只用来测试读和写的workload例子

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<workload name="s3-sample" description="sample benchmark for s3">

  <storage type="s3" config="accesskey=<accesskey>;secretkey=<scretkey>;proxyhost=<proxyhost>;proxyport=<proxyport>;endpoint=<endpoint>" />

  <workflow>
    <!-- 创建桶 -->
    <workstage name="init">
      <work type="init" workers="1" config="cprefix=s3testqwer;containers=r(1,2)" />
    </workstage>
    <!-- 预先写数据 -->
    <workstage name="prepare">
      <work type="prepare" workers="12" config="cprefix=s3testqwer;containers=r(1,2);objects=r(1,10000);sizes=c(64)KB" />
    </workstage>

    <!-- 开始测试 -->
    <!-- 读 -->
    <workstage name="read">
      <work name="read" workers="12" runtime="300">
        <operation type="read" ratio="100" config="cprefix=s3testqwer;containers=u(1,2);objects=u(1,10000)" />
      </work>
    </workstage>

    <!-- 写 -->
    <workstage name="write">
      <work name="write" workers="12" runtime="300">
        <operation type="write" ratio="100" config="cprefix=s3testqwer;containers=u(1,2);objects=u(10001,20000);sizes=c(64)KB" />
      </work>
    </workstage>

    <!-- 删除数据 -->
    <workstage name="cleanup">
      <work type="cleanup" workers="1" config="cprefix=s3testqwer;containers=r(1,2);objects=r(1,20000)" />
    </workstage>
    <!-- 删除桶 -->
    <workstage name="dispose">
      <work type="dispose" workers="1" config="cprefix=s3testqwer;containers=r(1,2)" />
    </workstage>

  </workflow>

</workload>
```

点击`submit new workloads`按钮上传已经修改好的xml文件

### 测试结果

1. Avg-ResTime  响应平均时间

2. Avg-ProcTime 平均处理时间

3. Throughput：吞吐量，也就是我们常说的TPS

4. bandwith：带宽

5. succ-ratio :成功数

### 其他性能测试工具

华为开源的obscmdbench https://github.com/huaweicloud-obs/obscmdbench


## 参考资料

1. [https://amito.me/2018/Ceph-Cluster-Performance-Benchmark/#%E8%8E%B7%E5%8F%96%E5%9F%BA%E5%87%86%E6%80%A7%E8%83%BD%E7%BB%9F%E8%AE%A1%E6%95%B0%E6%8D%AE](https://amito.me/2018/Ceph-Cluster-Performance-Benchmark/#获取基准性能统计数据)
2. https://ivanzz1001.github.io/records/post/ceph/2017/07/28/ceph-benchmark
3. https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/1.3/html/administration_guide/benchmarking_performance
4. https://github.com/intel-cloud/cosbench/blob/master/COSBenchUserGuide.pdf
5. https://www.cnblogs.com/landhu/p/5843282.html