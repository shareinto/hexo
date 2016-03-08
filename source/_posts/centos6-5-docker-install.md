title: centos6.5上安装docker环境
date: 2015-09-11 17:53:13
categories: docker
tags: 
  - centos
  - linux
  - docker
---
## 准备
docker容器最早受到RHEL完善的支持是从最近的CentOS 7.0开始的，官方说明是只能运行于64位架构平台，内核版本为2.6.32-431及以上（建议升级到3.10.x,否则低版本会出现各种各样莫名的问题），升级内核请参考[centos6.5内核升级](http://shareinto.github.io/2015/09/11/centos6-5-kernel-update/)

## 1. 禁用selinux
```bash
$ getenforce
Enforcing

$ setenforce 0

$ vi /etc/selinux/config

SELINUX=disabled
```

## 2. 安装 Fedora EPEL
```bash
$ yum -y install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

## 3. 安装 docker-io

```bash
$ yum install docker-io -y

Dependencies Resolved

===========================================================================================
 Package                        Arch               Version          Repository     Size
Installing:
 docker-io                      x86_64         1.1.2-1.el6          epel          4.5 M
Installing for dependencies:
 lua-alt-getopt                 noarch         0.7.0-1.el6          epel          6.9 k
 lua-filesystem                 x86_64         1.4.2-1.el6          epel           24 k
 lua-lxc                        x86_64         1.0.6-1.el6          epel           15 k
 lxc                            x86_64         1.0.6-1.el6          epel          120 k
 lxc-libs                       x86_64         1.0.6-1.el6          epel          248 k
===========================================================================================
Install       6 Package(s)
```

## 4. 安装device-mapper相关包
```bash
$ yum install http://mirror.centos.org/centos/6/os/x86_64/Packages/device-mapper-1.02.95-2.el6.x86_64.rpm 
http://mirror.centos.org/centos/6/os/x86_64/Packages/device-mapper-libs-1.02.95-2.el6.x86_64.rpm 
http://mirror.centos.org/centos/6/os/x86_64/Packages/device-mapper-event-1.02.95-2.el6.x86_64.rpm 
http://mirror.centos.org/centos/6/os/x86_64/Packages/device-mapper-event-libs-1.02.95-2.el6.x86_64.rpm
```

## 5. 安装docker进程查看工具

```bash
$ yum install util-linux -y
$ wget --no-check-certificate -P ~ https://github.com/yeasy/docker_practice/raw/master/_local/.bashrc_docker;
$ echo "[ -f ~/.bashrc_docker ] && . ~/.bashrc_docker" >> ~/.bashrc; source ~/.bashrc
```

## 6. 安装完成,启动docker服务
```bash
$ service docker start
```

## 7. 续==》docker-compose安装
```bash
$ curl -L https://github.com/docker/compose/releases/download/1.2.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```