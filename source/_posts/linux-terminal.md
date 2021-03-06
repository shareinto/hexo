title: linux的控制终端
date: 2016-11-17 18:55:08
categories: linux
tags:
  - linux
  - 操作系统
------
# 问题
&#160; &#160; &#160; &#160;我们都知道，linux打开一个终端运行一个程序，在程序运行未结束的时候如果关掉终端的话，那么该程序也会跟着退出。但我们有时候需要长期间的运行一个程序，又不想开着我们的终端，那么我们就可以利用下面的语句
```bash
$ nohup <command> &
```
- nohup的意思是让我们的程序进程忽略所有挂断（SIGHUP）信号。
- "&" 符号表示让我们的程序进程进入后台运行

为什么这样做以后我们的程序就不会退出呢，接下来我们先补充一些相关的知识

# 基础知识

## 终端
&#160; &#160; &#160; &#160;终端(Terminal)也是一台物理设备，只用于输入输出，本身没有强大的计算能力。一台计算机只有一个控制台，在计算资源紧张的时代，人们想共享一台计算机，可以通过终端连接到计算机上，将指令输入终端，终端传送给计算机，计算机完成指令后，将输出传送给终端，终端将结果显示给用户。

## 登录终端
&#160; &#160; &#160; &#160;在早期的计算机上面，用户用哑终端（用硬连接连到主机）进行登录，这种登录要经由内核的终端设备驱动程序。因为连到主机上的终端设备数是固定的，所以同时的登录数也就有了已知的上限。随着图形终端的出现，创建终端窗口的应用也被开发出来，它仿真了基于字符的终端，使用户可以用熟悉的方式（shell命令行）与主机进行交互。包括使用网络进行远程登录的远程终端也是使用的这种方式。

## 伪终端
&#160; &#160; &#160; &#160;随着图形终端的出现，创建终端窗口的应用也被开发出来，它仿真了基于字符的终端，使用户可以用熟悉的方式（shell命令行）与主机进行交互。包括使用网络进行远程登录的远程终端也是使用的这种方式。网络登录与传统的串行终端登录的区别在于，前者必须等待一个网络连接请求到达，而不是使一个进程等待每一个可能的登录。为了使同一个软件既能处理终端登录又能处理网络登录，系统使用了一种称为伪终端（pseudo terminal）的软件驱动程序，
它仿真串行终端的运行行为，并将终端操作映射为网络操作。

## 进程组
&#160; &#160; &#160; &#160;每个进程除了有一进程ID外，还属于一个进程组，进程组就是一个或多个进程的集合。
&#160; &#160; &#160; &#160;那为啥Linux里要有进程组呢？其实，提供进程组就是为了方便对进程进行管理。假设要完成一个任务，需要同时并发100个进程。当用户处于某种原因要终止 这个任务时，要是没有进程组，就需要手动的一个个去杀死这100个进程。

## 作业
&#160; &#160; &#160; &#160;Shell分前后台来控制的不是进程而是作业（Job）或者进程组（Process Group）。一个前台作业可以由多个进程组成，一个后台也可以由多个进程组成，Shell可以运行一个前台作业和任意多个后台作业，这称为作业控制。
&#160; &#160; &#160; &#160;作业与进程组的区别：如果作业中的某个进程又创建了子进程，则子进程不属于作业。一旦作业运行结束，Shell就把自己提到前台，如果原来的前台进程还存在（如果这个子进程还没终止），它自动变为后台进程组。

## 会话
&#160; &#160; &#160; &#160;会话（session）是一个或多个进程组的集合。
![会话](/image/session.jpg)
&#160; &#160; &#160; &#160;如图，该会话中有三个进程组。通常是由shell的管道将几个进程编成一组的。上图有可能是由下列形式的shell命令形成的:
```bash
$ proc1 | proc2 & 
$ proc3 | proc4 | proc5
```

## 控制终端
- 当一个终端与一个会话相关联后，那么这个终端就称为该会话的控制终端(controlling terminal)。
- 建立与控制终端连接的会话首进程被称为控制进程(controlling process)。
- 一个会话中的几个进程组可被分成一个前台进程组(foreground process group)以及一个或多个后台进程组(background process group)。
- 如果一个会话有一个控制终端的话， 则它有一个前台进程组，其他进程组为后台进程组。
- 无论何时键入终端的中断键或退出键，都会将中断信号或退出信号发送至前台进程组的所有进程。
- 如果终端检测到调制解调器（或网络）断开，则挂断信号（SIGHUP）发送至控制进程（会话首进程），如果会**话首进程退出,则将挂断信号（SIGHUP）发送至前台进程组的所有进程**。
![会话](/image/session2.jpg)
&#160; &#160; &#160; &#160;有的时候程序的标准输入，输出会被重定向到其它地方，那么会话中的进程要**获取终端的话可以open文件/dev/tty**，这就告诉内核我要获取当前会话的控制终端。如果会话没有控制终端的话，那么对此设备的open将失败。

## 孤儿进程组

&#160; &#160; &#160; &#160;孤儿进程组定义为：该组中每个成员的父进程要么是该组的一个成员，要么不是该组所属会话的成员。换句话说，一个进程组中有一个进程，其父进程在属于同一个会话的另一个组中，那么它就不是孤儿进程组。
>在POSIX.1中，要求向新产生的孤儿进程组中处于停止状态的每一个进程发送挂断信号（SIGHUP），接着又向其发送继续信号（SIGCONT）。**对挂断信号的系统默认动作是终止该进程**。

# 问题分析

1. 当控制中端退出的时候，首先会发一个挂断信号(SIGHUP)给会话首进程，一般会话首进程都是shell进程，而此动作将导致shell进程退出。
2. 当会话首进程退出时，挂断信号（SIGHUP）还会继续发送给前台进程组的所有进程。
3. 若进程未对挂断信号（SIGHUP）进行处理时，内核默认动作是终止该进程。

&#160; &#160; &#160; &#160;由此我们可以得出，导致进程退出的罪魁祸首就是这一个挂断信号（SIGHUP）。回到文章最前面提出的解决方案：
```bash
$ nohup <command> &
```
&#160; &#160; &#160; &#160;其中的nohup命令是让程序忽略**所有**的挂断信号，所以就算终端退出，我们的程序也不会退出了。而最后面的"&"符号其实对防止程序退出没有任何用处。

# 深入
&#160; &#160; &#160; &#160;那么我们能不能让程序自动具备防退出功能呢？其实原理很简单，我们只要注册挂断信号(SIGHUP)的处理程序，那么内核就认为你对该信号作出响应了，自然就不会终止程序了。来看代码：
```cpp
 #include "apue.h"
 #include <fcntl.h> 
 static void sig_hup(int signo)
 {
   printf("received signup\n");
 }  
 static int count; 
 int main(void)
 {
   setbuf(stdout,NULL);
   printf("pid = %d\n",getpid());
   signal(SIGHUP, sig_hup);
   while(1)
   { 
     count++;
   } 
   exit(0);
 } 
```
&#160; &#160; &#160; &#160;函数void (*signal(int, void(*)(int)))(int) 带两个参数,一个为整型,一个为函数指针，返回值也是一个函数指针。这两个函数指针所指向的函数接受一个整型参数 且没有返回值。我们代码中定义的sig_hup正好是这一种类型。因此可以将其注册为挂断信号（SIGHUP）的处理函数。这样一来，我们的程序就不会因收到挂断信号而退出了。

# 思考

&#160; &#160; &#160; &#160;如果我们再次向进程发送SIGHUP信号(可利用kill -1命令)，那么我们的程序将会怎样呢，对于这种状况我们又该如何应对呢？

(全文完)