# 升级内核

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist

yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*

yum --enablerepo=elrepo-kernel install kernel-lt -y

yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64  -y

yum --enablerepo=elrepo-kernel install kernel-lt-tools -y

grub2-set-default 0   # 设置第一个为默认启动项
grub2-editenv list
```

