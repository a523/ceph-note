# 部署块

## 创建 RBD 存储池

```shell
ceph osd pool create pool_name pg_num pgp_num replicated crush_ruleset_name \
expected_num_objects
```

## 关联应用

```shell
ceph osd pool application enable pool_name application_name
```

```shell
rbd pool init <pool-name>
```

## 创建块设备用户（可选）

## 创建块设备映像

```shell
rbd create --size 1024 foo
```

## 重设映像大小

```shell
rbd resize --size 2048 foo 
```

## 删除映像

```shell
rbd rm {image-name}
```

## 快照



## 禁止特性

```shell
rbd feature disable rbd/bar object-map fast-diff deep-flatten
```

​	

