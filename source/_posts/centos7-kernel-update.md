title: centos7内核升级
date: 2018-09-17 17:27:31
categories: linux
tags: 
  - linux 
  - centos 
  - kernel
---
## 1. 查看当前内核版本
```bash
$ uname -r
3.10.0-229.el7.x86_64
```

## 2. 导入public key
```bash
$ rpm --import http://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

## 3. 安装ELRepo到CentOS-6.5中
```bash
$ rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

## 4. 安装**kernel-lt（lt=long-term）**
```bash
$ yum --enablerepo=elrepo-kernel install kernel-lt -y
```

## 5. 修改Grub引导顺序
```bash
$ grub2-set-default 0
```

## 6. 重启，查看内核版本号
```bash
$ uname -r
4.4.156-1.el7.elrepo.x86_64
```
