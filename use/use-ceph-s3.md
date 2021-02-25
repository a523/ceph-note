# ceph 对象存储的使用

ceph 对象存储兼容两种接口（协议）S3 和 swift.  S3 最常用， 这里仅介绍如何连接（使用）S3存储接口。

1. [命令行的方式（s3cmd）](#一s3cmd)
2. [熟悉的文件挂载方式  (s3fs)](#二像文件一样的用对象存储s3fs)
3. [GUI](#三gui工具)
4. [SDK (最常用的方式)](#四sdk)

**访问 S3, 你需要 S3 的:**

- 访问地址（endpoint）如：http://192.168.1.100/
- 用户密钥，包含：access_id 和 secret_key.  access_id是整个集群唯一的，可以根据access_id确定一个用户
- 具体的桶名（或者你有自己新建桶的权限） 桶名也是整个集群唯一的。（也就是你的桶不能和自己以及*其他用户*的桶重名）

> 获取以上资料，请参考[管理文档](../admin_and_ops/s3.md)，或者联系管理员。

## 一、s3cmd

### 参考资料：

1. https://s3tools.org/s3cmd-howto
2. https://docs.jdcloud.com/cn/object-storage-service/s3cmd

### 安装

```shell
yum install s3cmd -y
```

请尽量使用**新版**s3cmd

```shell
[xin@centos5 ~]$ s3cmd --version
s3cmd version 2.1.0
```

### 配置

```shell
s3cmd --configure
```

运行该命令，会打开一个交互模式，邀请你输入用户密钥和网关访问地址等

最终会根据你的输入生成配置文件在`~/.s3cfg`， 你也可以直接编辑该配置文件。

（当你通过交互式命令不能配置成功的时候，可以尝试直接编辑该配置文件，设置signature_v2 = True）

配置文件参考示例如下：  

```shell
[xin@centos5 ~]$ cat ~/.s3cfg 
[default]
access_key = 1ND9LJBZ2NNZG2759UXD  # 密钥
access_token = 
add_encoding_exts = 
add_headers = 
bucket_location = US
ca_certs_file = 
cache_file = 
check_ssl_certificate = True
check_ssl_hostname = True
cloudfront_host = cloudfront.amazonaws.com
connection_pooling = True
content_disposition = 
content_type = 
default_mime_type = binary/octet-stream
delay_updates = False
delete_after = False
delete_after_fetch = False
delete_removed = False
dry_run = False
enable_multipart = True
encrypt = False
expiry_date = 
expiry_days = 
expiry_prefix = 
follow_symlinks = False
force = False
get_continue = False
gpg_command = /usr/bin/gpg
gpg_decrypt = %(gpg_command)s -d --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_encrypt = %(gpg_command)s -c --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_passphrase = 
guess_mime_type = True
host_base = 192.168.22.104:8080  # 网关访问地址
host_bucket = 192.168.22.104:8080/%(bucker)s  # 桶的访问形式（有两种：1.path 风格，2.二级域名风格，这里是path风格）
human_readable_sizes = False
invalidate_default_index_on_cf = False
invalidate_default_index_root_on_cf = True
invalidate_on_cf = False
kms_key = 
limit = -1
limitrate = 0
list_md5 = False
log_target_prefix = 
long_listing = False
max_delete = -1
mime_type = 
multipart_chunk_size_mb = 15
multipart_max_chunks = 10000
preserve_attrs = True
progress_meter = True
proxy_host = 
proxy_port = 0
public_url_use_https = False
put_continue = False
recursive = False
recv_chunk = 65536
reduced_redundancy = False
requester_pays = False
restore_days = 1
restore_priority = Standard
secret_key = BKXKqgybF6KdwDIlb4AhpdpXFdvoTVWDS79QgcHt  # 密钥
send_chunk = 65536
server_side_encryption = False
signature_v2 = False
signurl_use_https = False
simpledb_host = sdb.amazonaws.com
skip_existing = False
socket_timeout = 300
stats = False
stop_on_error = False
storage_class = 
throttle_max = 100
upload_id = 
urlencoding_mode = normal
use_http_expect = False
use_https = False  # 有没有用https， 根据实际情况修改
use_mime_magic = True
verbosity = WARNING
website_endpoint = http://%(bucket)s.s3-website-%(location)s.amazonaws.com/
website_error = 
website_index = index.html
```

### 使用

```shell
s3cmd ls # 列出该用户下的所有桶

s3cmd mb s3://{new_bucket}  # 创建新桶
```

### 上传文件

```shell
[xin@centos5 ~]$ s3cmd put a_file s3://repo
upload: 'a_file' -> 's3://repo/a_file'  [1 of 1]
 7 of 7   100% in    0s   128.02 B/s  done
```

更多使用请参考上文参考资料



## 二、像文件一样的用对象存储（s3fs）

参考资料：

1. https://aws.amazon.com/cn/blogs/china/s3fs-amazon-ec2-linux/

2. https://github.com/s3fs-fuse/s3fs-fuse

### 安装s3fs

```shell
sudo yum install epel-release
sudo yum install s3fs-fuse
```

### 配置密钥

```
echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ${HOME}/.passwd-s3fs
chmod 600 ${HOME}/.passwd-s3fs
```

### 挂载

语法

```shell
s3fs yourbucket /path/to/mountpoint -o passwd_file=${HOME}/.passwd-s3fs -o url=https://url.to.s3/ -o use_path_request_style
```

例如

```shell
mkdir ~/data
s3fs bucketname ~/data -o passwd_file=${HOME}/.passwd-s3fs -o url=http://192.168.22.99:8899/ -o use_path_request_style -o createbucket
# 如果你的桶之前没有创建好，记得加选项 -o createbucket， s3fs会帮你创建
```

### 验证

```shell
[xin@centos1 ~]$ df -h
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 475M     0  475M    0% /dev
tmpfs                    487M     0  487M    0% /dev/shm
tmpfs                    487M  7.7M  479M    2% /run
tmpfs                    487M     0  487M    0% /sys/fs/cgroup
/dev/mapper/centos-root   17G  3.3G   14G   19% /
/dev/sda1               1014M  167M  848M   17% /boot
tmpfs                    487M   24K  487M    1% /var/lib/ceph/osd/ceph-0
tmpfs                     98M     0   98M    0% /run/user/1000
s3fs                     256T     0  256T    0% /home/xin/data
```

看到最下一行内容，可以看到我们挂载的目录，文件系统是 s3fs

### 可选参数

-o multipart_size 10: 分块大小10M

-o parallel_count 10 : 10线程并发

-o multireq_max (default="20")：列出对象时的线程数

-d s3fs 的日志会写到系统日志中 ， 两个 -d 输出日志到终端

-o use_cache (默认=“”表示已禁用），指向本地文件夹用作缓存

-o umask=000,uid=1000,gid=1000 , 设置挂载到本地文件后权限，如果不正确设置，可能会出现权限问题

-o notsup_compat_dir 不对比目录，如果你只用 s3fs 上传对象到该桶，开启此选项，可提升写性能（**推荐**）

-o use_path_request_style  使用path风格的请求url，（目前集群只支持这种方式，所以这个选项必须加上）

-o dbglevel=info  日志等级

```shell
s3fs sources /data -o passwd_file=${HOME}/.passwd-s3fs -o url=http://192.168.8.54/ -o use_path_request_style  -o parallel_count=128  -o multireq_max=600 -oreadwrite_timeout=600 -oconnect_timeout=360 -odefault_acl=public-read -o notsup_compat_dir -o umask=022,uid=1006,gid=1006 -f
```

查看当前用户ID

```shell
id
```

更多使用（如实现开机自动挂载）请参考上文参考资料

## 三、GUI工具

1. [Cyberduck](https://cyberduck.io/)       Mac | Windows |  Linux
2. [s3browser](https://s3browser.com/download.aspx)    Windows
3. [CloudBerry](https://www.msp360.com/explorer.aspx)     Windows | Mac
4. 还有很多App 都支持 s3 接口， 如 FileZilla Pro

## 四、SDK

虽然前面介绍了这么多使用对象存储的方法， 但其实在实际应用中，用对象存储最多的方式是使用SDK。

SDK 几乎覆盖了常用的编程语言。

ceph 的对象存储兼容 s3 接口，你可以从下面的页面中选择一个你喜欢编程语言的SDK。

https://aws.amazon.com/cn/s3/developer-resources/?nc=sn&loc=6#SDKs

根据SDK的文档，开始使用对象存储。

个人理解，对象存储接口是为互联网应用设计的， 而不是为最终用户设计的， 所以对应用访问友好。最终用户使用的通常是开发好的产品。
