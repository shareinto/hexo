title: centos6.5内核升级
date: 2015-09-11 17:27:31
categories: linux
tags: 
  - linux 
  - centos 
  - kernel
---
## 1. 查看当前内核版本
```bash
$ uname -r
2.6.32-431.el6.x86_64
```

## 2. 导入public key
```bash
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

## 3. 安装ELRepo到CentOS-6.5中
```bash
$ rpm -ivh http://www.elrepo.org/elrepo-release-6-5.el6.elrepo.noarch.rpm
```

## 4. 安装**kernel-lt（lt=long-term）**
```bash
$ yum --enablerepo=elrepo-kernel install kernel-lt -y
```

## 5. 编辑grub.conf文件，修改Grub引导顺序
```bash
$ vim /etc/grub.conf

# grub.conf generated by anaconda
#
# Note that you do not have to rerun grub after making changes to this file
# NOTICE:  You do not have a /boot partition.  This means that
#          all kernel and initrd paths are relative to /, eg.
#          root (hd0,0)
#          kernel /boot/vmlinuz-version ro root=/dev/sda1
#          initrd /boot/initrd-[generic-]version.img
#boot=/dev/sda
default=0
timeout=5
splashimage=(hd0,0)/boot/grub/splash.xpm.gz
hiddenmenu
title CentOS (3.10.28-1.el6.elrepo.x86_64)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.28-1.el6.elrepo.x86_64 ro root=UUID=0a05411f-16f2-4d69-beb0-2db4cefd3613 rd_NO_LUKS  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_MD crashkernel=auto LANG=en_US.UTF-8
 rd_NO_LVM rd_NO_DM rhgb quiet
        initrd /boot/initramfs-3.10.28-1.el6.elrepo.x86_64.img
title CentOS (2.6.32-431.3.1.el6.x86_64)
        root (hd0,0)
        kernel /boot/vmlinuz-2.6.32-431.3.1.el6.x86_64 ro root=UUID=0a05411f-16f2-4d69-beb0-2db4cefd3613 rd_NO_LUKS  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_MD crashkernel=auto LANG=en_US.UTF-8 r
d_NO_LVM rd_NO_DM rhgb quiet
        initrd /boot/initramfs-2.6.32-431.3.1.el6.x86_64.img
title CentOS (2.6.32-431.el6.x86_64)
        root (hd0,0)
        kernel /boot/vmlinuz-2.6.32-431.el6.x86_64 ro root=UUID=0a05411f-16f2-4d69-beb0-2db4cefd3613 rd_NO_LUKS  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_MD crashkernel=auto LANG=zh_CN.UTF-8 rd_NO
_LVM rd_NO_DM rhgb quiet
        initrd /boot/initramfs-2.6.32-431.el6.x86_64.img
```
>确认刚安装好的内核在哪个位置，然后设置default值（从0开始），一般新安装的内核在第一个位置，所以设置default=0。

## 6. 重启，查看内核版本号
```bash
$ uname -r
3.10.28-1.el6.elrepo.x86_64
```