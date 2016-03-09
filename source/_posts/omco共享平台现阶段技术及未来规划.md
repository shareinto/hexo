title: omco共享平台现阶段技术及未来规划
date: 2016-03-08 16:20:48
tags:
  - omco
  - sdp
---
# 现阶段omco共享平台具备的技术
 1. 支持一键创建web应用和web组件:
   其中包括，git源码管理仓库的创建，jenkins项目创建，docker容器的部署。
这里docker容器的创建与管理用到了saltstack运维管理工具（考虑用kubernetes替换）

 2. 拥有自己的私有nexus仓库  

 3. 拥有gitlab源码管理仓库  

 4. 拥有ldap用户认证系统  

 5. 拥有docker镜像中心  

# 近期的第一个目标主要是要能满足水机线上业务的稳定性，这一点先满足以后，再考虑下一步的自动化或半自动伸缩部署功能:
## 关于稳定性，主要从以下几点着手：
1. centos6.5对network namespace支持不好，内核必须升级至2.6.32－504 或以上。

2. CentOS6.5 自带的 device mapper 存在 dm-thin discard 导致内核可能随机crash,解决的办法是关闭 discard
support ， 在 docker 配 置 中 添 加 “--storage-opt dm.mountopt=nodiscard --storage-opt
dm.blkdiscard=false”，并且严格禁止磁盘超配，因为磁盘超配可能会导致整个 device
mapper 无法分配磁盘空间，而把整个文件系统变成只读，从而引起严重问题。

3. 在宿主机上增加多维度的监控，包括内存，cpu，io，进程存活性，内核日志，实时pid数，网络连接跟踪数等。

4. 灾备：主要是要解决一键跨物理机的迁移。

5. 调整  vm.dirty_expire_centisecs,vm.dirty_writeback_centisecs, vm.extra_free_kbytes 等内核参数。

6. 合并镜像层，来减少镜像pull的时间

7. registry镜像中心性能优化

8. 更换docker镜像集群管理方案为kubernetes