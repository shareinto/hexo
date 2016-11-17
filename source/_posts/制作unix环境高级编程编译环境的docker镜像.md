title: 制作unix环境高级编程编译环境的docker镜像
date: 2016-10-31 20:46:34
categories: linux
tags: 
  - docker
  - linux
  - 操作系统
---
## 准备
首先，下载第三版的[源码包](http://www.apuebook.com/src.3e.tar.gz)

再下载两个libbsd库
[http://elrepo.reloumirrors.net/testing/el6/x86_64/RPMS/libbsd-0.2.0-4.el6.elrepo.x86_64.rpm](http://elrepo.reloumirrors.net/testing/el6/x86_64/RPMS/libbsd-0.2.0-4.el6.elrepo.x86_64.rpm)
[http://elrepo.reloumirrors.net/testing/el6/x86_64/RPMS/libbsd-devel-0.2.0-4.el6.elrepo.x86_64.rpm](http://elrepo.reloumirrors.net/testing/el6/x86_64/RPMS/libbsd-devel-0.2.0-4.el6.elrepo.x86_64.rpm)

## 镜像build
### 1. 解压src.3e.tar.gz
>```bash
$ tar -zxvf src.3e.tar.gz
```
### 2. Dockerfile
>```bash
FROM centos:latest

MAINTAINER shareinto

COPY libbsd-0.2.0-4.el6.elrepo.x86_64.rpm /tmp/libbsd-0.2.0-4.el6.elrepo.x86_64.rpm

COPY libbsd-devel-0.2.0-4.el6.elrepo.x86_64.rpm /tmp/libbsd-devel-0.2.0-4.el6.elrepo.x86_64.rpm

RUN yum install make -y \
&& yum install gcc -y \
&& rpm -ivh /tmp/libbsd-0.2.0-4.el6.elrepo.x86_64.rpm \
&& rpm -ivh /tmp/libbsd-devel-0.2.0-4.el6.elrepo.x86_64.rpm \
&& rm -rf /tmp/libbsd-0.2.0-4.el6.elrepo.x86_64.rpm \
&& rm -rf /tmp/libbsd-devel-0.2.0-4.el6.elrepo.x86_64.rpm

COPY apue.3e /root/apue.3e

WORKDIR /root/apue.3e

RUN make \
&& cp ./include/apue.h ./lib/error.c /usr/include \
&& cp ./lib/libapue.a  /usr/lib \
&& rm -rf /root/apue.3e

WORKDIR /root
```
### 3. build
>```bash
$ docker build -t apue .
```
### 4. alias一下编译命令 方便编译
>```bash
$ vi ~/.bashrc
```
>添加下面到.bashrc里面
```bash
alias apue='apue(){ docker run --rm -v $(pwd):/root apue gcc $1 -o $(echo $1 | sed "s/\.c//")  -lapue; };apue'
```

>```bash
$ source ~/.bashrc
```

### 5. 编译
>```bash
$ apue ls.c 
```
>将会自动生成可执行文件ls