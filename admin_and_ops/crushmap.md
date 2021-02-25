# 编辑CRUSH Map

## 查看CRUSH map

### 方法一

```shell
ceph osd crush tree # 简单
ceph osd crush dump  # 详细
```

### 方法二

**如果上面显示的不明确，也可以：**

1. #### 获取 CRUSH 图
```shell
ceph osd getcrushmap -o {compiled-crushmap-filename}
```
2. #### 反编译CRUSH图
```shell
crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
```


有些时候你需要不同的存储池的数据存储到不同的存储介质上， 你得编辑CRUSH map

## 修改ceph.conf

### 方法一、 通过交互式命令修改
修改CRUSH map 前， 如果你要添加虚拟主机， 你需要在集群中设置如下选项，否则，您移动的 OSD 稍后将重设置到其在默认根中的原始位置，并且集群不会按预期方式工作。

在ceph.conf 加入如下选项：

```shell
osd crush update on start = false
```

并同步配置文件到所有节点。

#### 创建另一个root

```shell
ceph osd crush add-bucket ssd root
```

#### 创建虚拟节点

```shell
ceph osd crush add-bucket {bucket-name} {bucket-type}
# bucket-type 例如： host / rack
ceph osd crush move node1-ssd root=ssd
```

#### 移动OSD到虚拟节点

```shell
ceph osd crush add osd.0 1 root=ssd
ceph osd crush set osd.0 1 root=ssd host=node1-ssd
```



### 方法二、编译注入CRUSH map
用编辑打开之前导出的crush map
#### 修改完成之后编译
```shell
crushtool -c {decompiled-crush-map-filename} -o {compiled-crush-map-filename}
```
#### 注入
```shell
ceph osd setcrushmap -i  {compiled-crushmap-filename}
```

## 参考资料

1. https://documentation.suse.com/zh-cn/ses/5.5/html/ses-all/cha-storage-datamgm.html#op-mixed-ssd-hdd

