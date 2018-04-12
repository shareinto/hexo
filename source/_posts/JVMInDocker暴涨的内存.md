title: JVM-In-Docker:暴涨的内存
date: 2018-04-12 15:26:46
categories: linux
tags: 
  - java
  - docker
  - linux
------
# 引子
最近将java应用迁移至docker容器时，发现了一个有趣的现象：在容器内运行一段时间以后，总是被内核OOM Killed.顾名思义，这是java进程内存使用超过了一定的限制。
我们在启动容器的时候，限制其可用物理内存为1G，通过以下命令可查看内存限制

```
docker inspect --format='{{.HostConfig.Memory}}' ${ContainerId}
```
容器启动之后，我们能过grafana观察其内存变量，其内存使用在一小时之内就涨到了1G，再经过差不多一小时，容器就被OOM Killed了。
有趣的是，我们将该程序放到一以1G内存的虚拟机上去执行，其内存使用量一直保存在300M左右，自然也不会触发OOM Killed。
同样是1G内存的限制，为什么在容器内就挂了而在物理机或虚拟机上就正常了呢？看来这值得我们深挖一下。

# CGroup
首先，可以确定的是，容器之所以被杀死，是因为其内存超过了限制，那么这个限制，就是由Linux内核提供的CGroup来行限制的。
CGroup全名Control Group,其作用就是限制某一个或一组进程对系统资源的使用，在这里我们不对CGroup进行深入的探讨，有兴趣的同学可以[点击这里](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)进行深入学习。
在这里要提到的是，为什么上述容器在内存满了之后还要再过1小时才被Kill？这是因为我们的系统开启了交换内存的功能，而如果我们不对容器进行交换内存进行限制而只是限制物理内存，那么交换内存将会被设置为物理内存的2倍。（注意，这里交换内存包含了物理内存的容量）
所以当物理内存满了以后，容器还能继续跑是因为其还在涨的内存被交换出去了，还没到达交换内存的限制而已。

# JVM --Xmx
除非我们显示的设置JVM的最大堆大小，否则，JVM将会根据宿主机的RAM来推断这个值 -- 默认情况下，这个值会被设置成宿主机内存的1/4。
我们可以能过[-XX:+PrintFlagsFinal](http://matthewkwilliams.com/index.php/2015/10/02/looking-inside-a-jvm-xxprintflagsfinal/)来查看这个值的大小

笔者在一台64G内存的物理器上执行得到如下结果：
```
$ java -XX:+PrintFlagsFinal -version|grep -i heapsize|egrep 'Initial|Max'
java version "1.7.0_67"
Java(TM) SE Runtime Environment (build 1.7.0_67-b01)
Java HotSpot(TM) 64-Bit Server VM (build 24.65-b04, mixed mode)
    uintx InitialHeapSize                          := 1057991744      {product}           
    uintx MaxHeapSize                              := 16928210944     {product}
```
可以看到 MaxHeapSize为 ~16G
我们在容器中执行看一下结果：
```
$ docker run --rm java java  -XX:+PrintFlagsFinal -version |grep -i heapsize | egrep 'Initial|Max'
java version "1.7.0_67"
Java(TM) SE Runtime Environment (build 1.7.0_67-b01)
Java HotSpot(TM) 64-Bit Server VM (build 24.65-b04, mixed mode)
    uintx InitialHeapSize                          := 1057991744      {product}           
    uintx MaxHeapSize                              := 16928210944     {product}
```
一样的结果。现在我们限制一下容器的内存为 1G(-m 1024m)
```
$ docker run -m 1024m --rm java java  -XX:+PrintFlagsFinal -version |grep -i heapsize | egrep 'Initial|Max'
java version "1.7.0_67"
Java(TM) SE Runtime Environment (build 1.7.0_67-b01)
Java HotSpot(TM) 64-Bit Server VM (build 24.65-b04, mixed mode)
    uintx InitialHeapSize                          := 1057991744      {product}           
    uintx MaxHeapSize                              := 16928210944     {product}
```
结果还是一样。

# Memory Inside Linux Containers
我们用free来看一下内存的情况
在宿主机上:
```
$ free -m
             total       used       free     shared    buffers     cached
Mem:         64574      62979       1594          0        864      49236
-/+ buffers/cache:      12878      51695
Swap:         4095        448       3647
```
在容器内:
```
$ docker run -1024 --rm busybox free -m
             total       used       free     shared    buffers     cached
Mem:         64574      62979       1594          0        864      49236
-/+ buffers/cache:      12878      51695
Swap:         4095        448       3647
```
还是一模一样，为什么会出现这样的情况？这就等于我们在容器内无法读到容器的限制内存？
关于这个问题，这里有一篇文章（[Memory inside Linux containers](https://fabiokung.com/2014/03/13/memory-inside-linux-containers/)）解释得很详细,这里我们作一个简单的总结：
意思是我们并不是没办法读取容器内存的，而是读的地方不对。早期的统计工具，例如top,free（包括jvm）等都是从/proc目录下读取系统信息的，而在容器内，容器的内存限制并不是在/proc下面，而要从cgroup读取（docker内的cgroup目录在/sys/fs/cgroup),
而/proc目录，到目前为止，docker官方并没有对其进行容器化处理，所以从这里读取的信息都还是宿主机的信息，这也就很好解释我们上面看到的实验结果。

## 解决
1. 一些人认为可以提供一个用户态的库供大家调用，这种方案只适合新的程序，那些旧的程序如top,free甚至jvm都无法使用这个方案了
2. 另一种方案是在用户态重写/proc/meminfo，例如[FUSE](https://github.com/libfuse/libfuse)就可以实现这种功能。但这种方案的问题是要在宿主机上运行这个用户态的服务，如果这个服务挂了该如何处理？总之，这种方案可以解决问题，但并不是十分完美
3. 直接内核支持，但是这种又会影响到那些没有容器化的系统，所以除非你自定义内核，否则不可能被合并到主流内核版本中。

## JVM的解决
知道了问题产生的原因，那么我们解决JVM在容器中的问题其实就很简单了：
我们结合容器的限制内存，给JVM设置一下最大堆参数就可以了
```
$ docker run -m 1024m --rm java java -Xmx256m -Xms16m  -XX:+PrintFlagsFinal -version|grep -i heapsize|egrep 'Initial|Max'     
java version "1.7.0_67"
Java(TM) SE Runtime Environment (build 1.7.0_67-b01)
Java HotSpot(TM) 64-Bit Server VM (build 24.65-b04, mixed mode)
    uintx InitialHeapSize                          := 16777216        {product}           
    uintx MaxHeapSize                              := 268435456       {product}
```
可以看到MaxHeapSize=256m,InitialHeapSize=16m

笔者最后给最开始被OOM Killed的Java程序加上最大堆限制之后，就再也没有出现被Kill的情况，附上内存图：
![](http://7xlovv.com1.z0.glb.clouddn.com/JVMInDocker.jpg)

（全文完）