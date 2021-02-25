# 当前 ceph 集群中的角色

- ceph 0    192.168.8.49
  - mgr
  - rgw
  - 
- ceph 1   192.168.8.50
  - mon 
  - mgr
  - haproxy
  - Keepalived
- ceph 2   192.168.8.51
  -  mon
  - rgw
- ceph 3   192.168.8.52
  - mon
  - rgw





对象存储网关接口：http://192.168.8.54

haproxy web：http://192.168.8.50:8080/haproxy

Dashboard：https://192.168.8.50:8443

