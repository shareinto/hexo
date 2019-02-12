title: linux中的文件强制锁
date: 2016-12-07 17:39:44
categories: linux
tags:
  - linux
  - 操作系统
------
# 什么是建议性锁和强制性锁
- 建议性锁：建议性锁并不从内核限制程序访问文件，而是依赖各个合作进程(cooperating process)之间遵循相应的规则。
- 强制性锁：强制性锁会让内核检查每一个open、read、和write,验证调用进程是否违背了正访问的文件上的某一把锁。
>就好比红灯亮了，人们遵守规则不闯红灯，但是如果有人要强行闯红灯的话，并没有好的方法去阻止，这就是建议性锁。
>但是如果我们在红灯亮的同时，把道路也封起来，这时候你想闯也闯不了，这就是强制性锁。

# Linux上的强制性锁
Linux系统上的强制性锁默认情况下是不开启的。如果要开启强制性锁，要由以下两个步骤完成：
1. 在文件系统mount的时候加上-o mand参数
2. 打开文件的设置组ID位并且关闭其组执行位

在shell下可以这样打开
```bash
 $ chmod g+s <filename>
 $ chmod g-x <filename>
```
&#160; &#160; &#160; &#160;或者通过fchmod函数设置
```cpp
fchmod(fd,(statbuf.st_mode & ~S_IXGRP) | S_ISGID)
```

# 验证强制性锁

 我们可以编写一段测试程序，它打开一个文件（系统已打开强制性锁模式），对该文件整体设置一把读锁，然后休眠一段时间。
 该程序如下：
 ```cpp
 #include "apue.h"
 #include <fcntl.h>
 #include <stdio.h>
    
int main(int argc,char *argv[])
{
   int fd;
   pid_t pid;
   struct stat statbuf;
   if(argc != 2)
   {
     fprintf(stderr,"usage: %s filename\n",argv[0]);
     exit(1);
   }
   if((fd = open(argv[1],O_RDWR | O_CREAT | O_TRUNC,FILE_MODE)) < 0)
   {
     err_sys("open error");
   }
 
   if(write(fd,"abcef",6) != 6)
   {
     err_sys("write error");
   } 
   if(fstat(fd,&statbuf) < 0)
   {
     err_sys("fstat error");
   } 
   if(fchmod(fd,(statbuf.st_mode & ~S_IXGRP) | S_ISGID) < 0)
   {
     err_sys("fchmod error");
   }
   if((read_lock(fd, 0, SEEK_SET, 0)) < 0)
   {
     err_sys("read_lock error");
   }
   sleep(60);
   exit(0);
}
 ```
 运行程序
 ```bash
 $ ./lock temp.lock
 ```
 在另一个终端验证
 ```bash
 $ echo "hello" > temp.lock
 ```
 可以看到出现了下面的错误
 ```bash
 -bash: temp.lock: Resource temporarily unavailable
 ```
 事实证明我们的读锁生效了。
 
 # 绕过强制性锁
 我们用vi程序对temp.lock进行洗编辑，其结果竟然可以写回磁盘！强制性锁不起作用了？
 我们用strace -c vim 命令跟踪vim程序的系统调用
 ```bash
 $ strace -c vim temp.lock
 ```
 返回如下信息：
 ```bash
 % time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000015          15         1           setxattr
  0.00    0.000000           0        34           read
  0.00    0.000000           0        31           write
  0.00    0.000000           0        29         2 open
  0.00    0.000000           0        29           close
  0.00    0.000000           0        22         4 stat
  0.00    0.000000           0        20           fstat
  0.00    0.000000           0         4         3 lstat
  0.00    0.000000           0         2           poll
  0.00    0.000000           0         5           lseek
  0.00    0.000000           0        34           mmap
  0.00    0.000000           0        17           mprotect
  0.00    0.000000           0         8           munmap
  0.00    0.000000           0         6           brk
  0.00    0.000000           0        22           rt_sigaction
  0.00    0.000000           0         6           rt_sigprocmask
  0.00    0.000000           0        19           ioctl
  0.00    0.000000           0         6         2 access
  0.00    0.000000           0        46           select
  0.00    0.000000           0         1           getpid
  0.00    0.000000           0         2           socket
  0.00    0.000000           0         2           connect
  0.00    0.000000           0         2           sendto
  0.00    0.000000           0         1           recvmsg
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           uname
  0.00    0.000000           0         9           fcntl
  0.00    0.000000           0         1           fsync
  0.00    0.000000           0         8           getcwd
  0.00    0.000000           0         5           chdir
  0.00    0.000000           0         4           fchdir
  0.00    0.000000           0         1           rename
  0.00    0.000000           0         6         1 unlink
  0.00    0.000000           0         1         1 readlink
  0.00    0.000000           0         2           chmod
  0.00    0.000000           0         1           fchown
  0.00    0.000000           0         1           getrlimit
  0.00    0.000000           0         1           sysinfo
  0.00    0.000000           0         3           getuid
  0.00    0.000000           0         1           sigaltstack
  0.00    0.000000           0         1           statfs
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1         1 getxattr
------ ----------- ----------- --------- --------- ----------------
100.00    0.000015                   398        14 total
 ```
 我们可以发现其调用了rename函数，我们知道rename其实是通过unlink和link函数来实现对文件硬连接的改变。
 分析其原理：
 >vim将新内容写到一个临时文件中，然后删除原文件，最后将临时文件名改为原文件名。而强制性锁对unlink函数没有影响^_^。
 
 知道了原理那么我们可以自己编写一段代码来验证：
 ```cpp
#include "apue.h"
#include <fcntl.h>

int main(int argc,char *argv[])
{
  int fd;
  if(argc != 2)
  {
    fprintf(stderr,"usage: %s filename\n", argv[0]);
  }

  if((fd = open(".temp",O_RDWR | O_CREAT | O_TRUNC,FILE_MODE)) < 0)
  {
    err_sys("open error");
  }
  if(write(fd,"ghijkl",6) != 6)
  {
    err_sys("write error");
  }
  if(unlink(argv[1]) < 0)
  {
    err_sys("unlink error");
  }
  if(link(".temp",argv[1]) < 0)
  {
    err_sys("link error");
  }
  if(unlink(".temp") < 0)
  {
    err_sys("unlink error");
  }
  exit(0);
}
```
该程序先创建一个.temp临时文件，然后写放一些数据，接着unlink原文件，再将.temp重命名成原文件，最后记得unlink临时文件。
记得该程序的工作目录必须和原文件处在同一个磁盘，因为跨磁盘的link是不允许的。最后看看效果：
```bash
$ cat temp.lock
ghijkl
```

(全文完)