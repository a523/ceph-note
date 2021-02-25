# keepalived + LVS 负载均衡和高可用

## 名词

DS ： 负载均衡器的节点

RS： 后端真实节点

## 准备硬件节点

因为LVS RS节点需要修改内核参数， DS不需要。所以实际上RS 和DS的内核参数配置必须是不一样的， 所以我理解DS 和RS 不能在同一点节点。加上还要做高可以用的话， 你必须准备4个节点才有意义，即两个RS节点， 两个DS节点。

## 配置DS 节点

### 安装配置工具

```shell
yum install ipvsadm keepalived
```
1.开启服务器路由转发

```shell
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

### 修改配置文件

```shell
[cadmin@ceph1 my-cluster]$ cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived

global_defs {
   router_id ceph1
   vrrp_skip_check_adv_addr
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance RGW {
    state BACKUP
    interface em1
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.8.54/24   # 虚拟ip
    }
}


virtual_server 192.168.8.54 8080 {
    delay_loop 6
    lb_algo rr  # 负载均衡的算法
    lb_kind DR
    persistence_timeout 50
    protocol TCP
		# DR 模式， virtual_server 端口 和 real_server 端口必须一致

    real_server 192.168.8.51 8080 {  # 8080是RGW的端口号， 依据情况修改
        weight 1
        HTTP_GET {   # 写上如何判断真实服务正常
            url { 
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.8.52 8080 {
        weight 1
        HTTP_GET {
            url { 
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

```

其他可选配置说明

配置文件中的digest用genhash生成
```shell
[cadmin@ceph1 keepalived]$ genhash -p 8080 -s 192.168.8.51
MD5SUM = 1c37e94a515e3381a207d4403506e4b3
```
``` shell
nopreempt #不抢占
```

### 启动服务

```shell
sudo systemctl start keepalived 
```

## 配置RS节点

1. 忽略ARP响应
```shell
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
```
2. 添加虚拟ip到lo
```shell
ifconfig lo:0 $vip netmask 255.255.255.255  broadcast $vip up
```
也可以把上面操作脚本化，方便操作
```shell
cat ./rs.sh
#!/bin/bash
#description:Set vip of Real Server
if [ $# -eq 0 ];then
        echo "usage: $0 start/stop"
        exit 1
fi
viplist=('192.168.8.54')
mask=255.255.255.0
case $1 in
start)
        echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
        echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
        for vip in ${viplist[@]}
        do
                ifconfig lo:0 $vip netmask $mask  broadcast $vip up
        route add -host $vip dev lo:0
        done
;;
stop)
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce

        for vip in ${viplist[@]}
        do
                route del -host $vip dev lo:0
                ifconfig lo:0  down
        done

;;
esac
```
启用RS节点
```shell
sudo sh ./rs.sh start
```

## 测试

在非DR 节点也非RS节点访问虚拟ip机器对应服务的端口,  如：

```shell
curl 192.168.8.54:8080
```

正常情况下， 在RS节点访问VIP：port 也是可以访问业务的， 在DR 的备用节点也是可以访问， 但是DR是当前keepalived虚拟ip的所在节点便不行，

## 常见故障

- 可以ping通VIP，但是访问不了端口的服务

  LVS 会以虚拟ip去访问RS节点，请确保RS节点上的业务程序开放虚拟ip可以访问，你可以在RS节点上访问做测试

  ```shell
  curl vip:port
  ```

> 注意： 如果RGW 节点和 OSD 节点在同一个节点，目前用LVS做负载均衡的时候，因为需要在RS服务器配置VIP，看起来会触发一个bug https://tracker.ceph.com/issues/43413 ，对于这个现象，本人测试环境有复现。

> 防火墙不在本文讨论访问，如果你的环境有防火墙，记得修改相关设置


## 参考资料

1. https://support.huaweicloud.com/dpmg-kunpengwebs/kunpengwebs_04_0004.html
2. https://access.redhat.com/documentation/zh-tw/red_hat_enterprise_linux/7/html/load_balancer_administration/index
3. [http://docs.linuxtone.org/ebooks/LoadBalance/lvs/keepalived%20the%20definitive%20guide--FinalBSD.pdf](http://docs.linuxtone.org/ebooks/LoadBalance/lvs/keepalived the definitive guide--FinalBSD.pdf)
4. [https://keepalived-doc.readthedocs.io/zh_CN/latest/Keepalived%E9%85%8D%E7%BD%AE%E7%AE%80%E4%BB%8B.html](https://keepalived-doc.readthedocs.io/zh_CN/latest/Keepalived配置简介.html)
5. https://www.linuxprobe.com/keepalived-lvs-dr.html

