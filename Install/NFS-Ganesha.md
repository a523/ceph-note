# NFS-Ganesha
> 以centos7.4为例

## install 
安装rpcbind
```shell
sudo yum install nfs-utils
sudo systemctl start rpcbind
sudo systemctl enable rpcbind
```

添加源

```shell
[xin@centos1 ~]$ cat /etc/yum.repos.d/nfs-ganesha.repo 
[nfs-ganesha]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/nfs-ganesha/rpm-V3.3-stable/octopus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.ustc.edu.cn/ceph/keys/release.asc
priority=2
```

安装nfs-ganesha-rgw

```shell
sudo yum install nfs-ganesha-rgw
```

## config

Ganesha 节点需要和RGW在同一个节点， 可以访问RGW的密钥

```shell
[xin@centos6 /]$ cat etc/ganesha/ganesha.conf 

EXPORT
{
        ## Export Id (mandatory, each EXPORT must have a unique Export_Id)
        Export_Id = 2;

        ## Exported path (mandatory)
        Path = /;

        ## Pseudo Path (required for NFSv4 or if mount_path_pseudo = true)
        Pseudo = /;

        ## Restrict the protocols that may use this export.  This cannot allow
        ## access that is denied in NFS_CORE_PARAM.
        Protocols = 4;

        ## Access type for clients.  Default is None, so some access must be
        ## given. It can be here, in the EXPORT_DEFAULTS, or in a CLIENT block
        Access_Type = RW;

        ## Whether to squash various users.
        Squash = No_Root_Squash;

        ## Allowed security types for this export
        Sectype = sys;
        Transport_Protocols = TCP;

        ## Exporting FSAL
        FSAL {
                Name = RGW;
                User_Id = xin;
                Access_Key_Id = PORI9OOCJJNZQOE9PLTW;
                Secret_Access_Key = miFpkSfhREkz8j6YQO8qrUtwFONy5aqooIOXAmFB;
        }
}

NFSV4 {
    Allow_Numeric_Owners = true;
    Only_Numeric_Owners = true;
}

RGW {
    name = "client.rgw.centos7";
    cluster = "ceph";
    ceph_conf = "/etc/ceph/ceph.conf";
    #init_args = "--{arg}={arg-value}";
}
## Configure logging.  Default is to log to Syslog.  Basic logging can also be
## configured from the command line
LOG {
        ## Default log level for all components
        Default_Log_Level = WARN;

        ## Configure per-component log levels.
        Components {
                FSAL = INFO;
                NFS4 = EVENT;
        }

        ## Where to log
        Facility {
                name = FILE;
                destination = "/var/log/ganesha.log";
                enable = active;
        }
}
```

## 启动服务

```shell
sudo systemctl start nfs-ganesha
sudo systemctl enable nfs-ganesha
```

## 客户端配置

```shell
sudo yum install nfs-utils
sudo systemctl enable rpcbind
sudo systemctl start rpcbind
```

### 挂载

```shell
 mount -o vers=4 <IP ADDRESS>:/export /mnt
```

## 当前问题

- 当挂载的桶里面有大量对象时， `ls` 会卡主。

- 写大文件时，会报IO错误？