# 删除集群

在开发环境中， 你可能需要删除集群重装

```shell
ansible-playbook  infrastructure-playbooks/purge-cluster.yml
```

然后删除OSD节点上的lvm

```shell
ceph-volume lvm zap /dev/sdb  --destroy
ceph-volume lvm zap /dev/sdc  --destroy
...
# 补充
lvdisplay
vgdisplay

dmsetup ls/remove
lvremove vgremove pvremove
```

```shell
# vgremove officevg
  Volume group "officevg" successfully removed
```


参考资料：

1. https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/logical_volume_manager_administration/vg_admin#VG_remove_PV

   