# 扩容

## 一、初始化设置

[安装系统， 升级内核， 参考之前的机器设置](./install_os.md)

## 二、部署 osd

[参考之前的部署教程](./use_ceph-deploy.md)

## 三、加入 crushmap

由于本集群自定义了 crush map，关闭了自动发现硬件并加入
所以需要手动加入 crush map
参见 [crushmap.md](../admin_and_ops/crushmap.md)

