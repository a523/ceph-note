# obscmdbench 测试性能

```shell
git clone https://github.com/huaweicloud-obs/obscmdbench.git
```

```shell
cd obscmdbench/
```

## 添加秘钥

```shell 
vim users.dat
```

```shell
accountName,accessKey,secretKey,
accountName1,accessKey1,secretKey1,
accountName2,accessKey2,secretKey2,
...
```



## 测试参数配置

vim config.dat

OSCs = 192.168.8.54   # 网关地址

Users = 1   # 用户数， 和users.dat对应

ThreadsPerUser = 64   # 线程数

bucketNameFixed = testbucket   # 桶名

ObjectSize = 65536  # 对象大小

 ObjectNamePrefix = object.test   # 对象名前缀

ObjectsPerBucketPerThread = 1000  # 上传的对象数

## 开始测试

```shell
sudo python run.py 201  # 上传对象
sudo python run.py 202  # 获取对象
# 该程序目前只支持python2
```

