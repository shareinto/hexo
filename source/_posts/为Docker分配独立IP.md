title: 为Docker分配独立IP
date: 2017-07-10 17:31:36
categories: docker
tags:
  - docker
------
# Docker的网络模型
  熟悉docker的人都知道，它有以下四种网络模式
1. host
2. container
3. none
4. bridge

>要理解Docker的网络，首先要发解的是Linux下面的network namespace。Linux Namespace是Linux提供的一种内核级别环境隔离的方法。其中network namepspace是六种隔离中的一种。
简单来说，如果将某一个进程的network namespace为设置为ns1，那么它将无法看到宿主机上（默认的名称空间下）的任何网络设备，路由规则，iptables,甚至是整个tcp/ip协议栈。在ns1下面创建的网络设备等等，在宿主机（默认的名称空间下）也同样看不到这些新创建的设备。这样，让用户感觉像是让我们的进程跑在了另外一个操作系统上面，仿佛我们新创建了一个操作系统环境。

了解了network namespace，我在再来了解docker的网络模式
1. host:
当使用host模式启动容器时，这个容器将不会创建自己的network namespace，而是和宿主机共用同一个。那么这样也就很好理解了，我们的进程创建的任何网络设备，监听的任何端口，宿主机都可以感知得到，也就是说，容器可以使用宿主机的ip和端口资源。
2. none:
使用none模式，Docker容器拥有自己的network Namespace，但是，并不为Docker容器进行任何网络配置。也就是说，这个Docker容器没有网卡、IP、路由等信息。需要我们自己为Docker容器添加网卡、配置IP等。该模式和host模式的一个重要的区别就是，none模式有自己的network namespace，而host模式没有。
3. container:
这个模式指定新创建的容器和已经存在的一个容器共享一个network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。
4. bridge:
bridge模式是docker的默认网络模式。当Docker进程启动时，会在主机上创建一个名为docker0的虚拟网桥，此主机上启动的Docker容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。从docker0子网中分配一个IP给容器使用，并设置docker0的IP地址为容器的默认网关。在主机上创建一对虚拟网卡veth pair设备，Docker将veth pair设备的一端放在新创建的容器中，并命名为eth0（容器的网卡），另一端放在主机中，以vethxxx这样类似的名字命名，并将这个网络设备加入到docker0网桥中。

# bridge网络模式
作为docker默认的网络模式，是最复杂也是运用最广的模式。我们先来看一下，在这种模式下，它的网格拓扑结构。
![docker-bridge](http://7xlovv.com1.z0.glb.clouddn.com/docker-bridge.png)
这里首先要讲解一下linux下虚拟网桥的概念：
- 虚拟网桥：
  首先，它的主体部分是一个二层交换机，但是奇怪的是，我们在宿主机上查看linux网络设备docker0的时候，它会有一个ip地址（作者主机上的docker0）
  ![docker0](http://7xlovv.com1.z0.glb.clouddn.com/docker0.png)
  稍微有点网络常识的人会知道，交换机是二层设备，是没有ip地址的。那么这个ip地址又是怎么来的呢。
  我们可以思考一下，假设你买了一个物理交换机回来以后，我们的主机要如何使用这个交换机？答案很简单，用一根网线将主机上的某一块网卡接到交换机的一个端口上面！是的，那么docker0设备上的ip实际上就是主机上连接交换机网卡的ip。
  所以，linux的虚拟网桥实际上包括三部分：
  - 一个L2的交换机
  - 一个主机的网卡
  - 一根连接以上两部分的网线

- veth pair
  了解了虚拟网桥，我们再来看一下另一个linux的虚拟网络设备： veth-pair
  它实际上是一对虚拟网卡,从一张veth网卡发出的数据包可以直接到达它的peer veth,两者之间存在着虚拟链路。也就是说，这种虚拟设备包括以下三部分：
  - 一个安装在主机上的网卡
  - 另一个安装在主机上的网卡
  - 一根连接这两个网卡的网线
  大家可能会觉得奇怪，这样的网络设备有什么用，数据从一个网卡出去，再从另外一块网卡进来？其实，这种网络设备有一个特点，就是两块网卡可以分别处于不同的network namespace。
  ![veth-pair](http://7xlovv.com1.z0.glb.clouddn.com/veth-pair.png)
  docker正是利用了这种特性，将其中的一块网卡添加到容器内部，另外一块留在宿主机上面，大家通过ifconfig命令在宿主机可以看到vethxxx这样的网络设备，但是这样的网络设备它是没有ip地址的。
  ![vethxxx](http://7xlovv.com1.z0.glb.clouddn.com/vethxxx.png)
  这又是为什么呢？这要回到上面提到的虚拟网桥。实际上这块网卡被添加到了docker0的交换机设备上，变成了该交换机上的一个端口，交换机的端口没有ip也就很正常了。
  我们可以通过brctl命令，将一个物理设备添加到一个虚拟网桥上面：
  ```bash
  # brctl addif docker0 vethae36b9b
  ```
  这个命令的意思是将vethae36b9b这个网络设备添加到docker0这个网桥上面。
  还可以查看已添加到网桥上面的设备
  ```bash
  # brctl show docker0
  bridge name     bridge id               STP enabled     interfaces
  docker0         8000.0242f2558144       no              vethae36b9b
  ```
到此，我们来理解bridge网络模式的拓扑结构就很简单了，这是不是非常像我们家庭网络的结构：一个个容器代表了家里的一台台计算机，而宿主机这时候变成了连接外网的路由器了。在这个子网内部的各个容器之间是可以互相访问的，容器可以访问外部网络，而外部网络要访问内部容器，就必须通过nat端口映射才行。

# 给docker容器分配一个和宿主机处于同一网段的ip
bridge网络有一个问题，就是多个容器要同时对外暴露服务时，会竞争宿主机上面的端口，导致端口资紧张的情况发生。那么我们能不能给docker分配一个和宿主机处于同一个网段的ip，这样，外部网络就可以直接访问该容器了呢?答案当然是可以，我们现在就利用上面的知识，来更改一下docker的网络拓扑结构。
![docker-bridge2](http://7xlovv.com1.z0.glb.clouddn.com/docker-bridge2.png)
这里我们为了避免连接不上宿主机，另外创建一个虚拟网桥br0
em1是宿主机上的网卡，它的ip为172.24.133.39/24。我们的做法很简单，将em1添加到docker0网桥上，然后将ip（172.24.133.39）设置给br0设备。
```bash
# brctl addbr br0
# brctl stp br0 off
# ifconfig br0 172.24.133.39/24 up
# brctl addif br0 em1
# ifconfig em1 0.0.0.0
# route add default gw 172.24.133.254 dev br0
```
创建一对veth pair,并将其中一个添加到br0中,另一个设置给容器（docker的network namespace的名称就是容器id)

```bash
# ip link add peerA type veth peer name peerB 
# brctl addif br0 peerA
# ip link set peerA up
# ip link set peerB netns ${container-pid}

```
然后，进入到容器中，将eth1的ip设置为172.24.133.253
```bash
# ip link set dev peerB name eth1 
# ip link set eth1 up
# ip addr add 172.24.133.253/24 dev eth1
# route add default gw 172.24.133.254 dev eth1
```
此时，我们就可以通过172.24.133.253这个ip直接访问容器了
（全文完）