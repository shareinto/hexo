title: docker-开启远程访问
date: 2016-03-03 16:26:24
categories: docker
tags:
  - docker
---
默认情况下，Docker守护进程会生成一个socket（/var/run/docker.sock）文件来进程本地进程通信，而不会监听任何端口，因此只能在本地使用docker客户端或者使用Docker API进行操作。 
如果想在其他主机上操作Docker主机，就需要让Docker守护进程监听一个端口，这样才能实现远程通信。

修改Docker服务启动配置文件，添加一个未被占用的端口号，重启docker守护进程。

centos用户
```bash
# vim /etc/sysconfig/docker
other_args="-H tcp://0.0.0.0:4243"
# service docker restart
```

ubuntu用户
```bash
# vim /etc/default/docker
DOCKER_OPTS="-H 0.0.0.0:4243"
# service docker restart
```

此时发现docker守护进程已经在监听4243端口，在另一台主机上可以通过该端口访问Docker进程了。

```bash
# docker -H IP:4243 images
```

但是我们却发现在本地操作docker却出现问题。
```bash
# docker images
FATA[0000] Cannot connect to the Docker daemon. Is 'docker -d' running on this host?
```

这是因为Docker进程只开启了远程访问，本地套接字访问未开启。我们修改配置，然后重启即可。

centos用户
```bash
# vim /etc/sysconfig/docker
other_args="-H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock"
# service docker restart
```

ubuntu用户
```bash
# vim /etc/default/docker
DOCKER_OPTS="-H 0.0.0.0:4243 -H unix:///var/run/docker.sock"
# service docker restart
```

现在本地和远程均可访问docker进程了。