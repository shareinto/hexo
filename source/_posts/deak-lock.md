title: 协同进程的死锁问题
date: 2016-12-14 15:25:01
categories: linux
tags:
  - linux
  - 操作系统
------
# 管道
&#160; &#160; &#160; &#160;要理解协同进程的话，首先要知道什么是管道。管道是UNIX系统当中IPC的最古老形式，所有的UNIX系统都提供这种通信机制。
管道是通过调用pipe函数创建的:

```cpp
#include <unistd.h>
int pipe(int fd[2]);
```
经由参数返回两个文件描述符：fd[0]为读而打开，fd[1]为写而打开。fd[1]的输出是fd[0]的输入。下图是管道的结构：

![管道](/image/pipe.jpg)

单个进程中的管道几乎没有任何用处。通常，进程会先调用pipe，接着调用fork，从而创建从父进程到子进程的IPC通道，反之亦然。下图显示了这种情况

![父子进程的管道](/image/fork_pipe.jpg)

fork之后做什么取决于我们想要的数据流的方向。对于从父进程到子进程的管道，父进程关闭管道的读端(fd[0]),子进程关闭写端(fd[1])。下图显示了在此之后描述符的状态结果：

![父子进程的管道](/image/fork_pipe2.jpg)

# 协同进程
&#160; &#160; &#160; &#160;当A进程既产生B进程的输入，又读取B进程的输出时，B进程就变成了A进程的**协同进程**（coprocess）。下图显示了这种安排：
![协同进程](/image/coprocess.jpg)

# 示例
让我们通过一个示例来观察协同进程。我们先创建一个简单的协同进程，它从其标准输入读取两个数，计算它们的和，然后将和写至其标准输出。
```cpp
#include "apue.h"

int main(void)
{
  int n, int1,int2;
  char line[MAXLINE];

  while((n = read(STDIN_FILENO, line, MAXLINE)) > 0)
  {
    line[n] = 0;
    if(sscanf(line, "%d%d",&int1,&int2) == 2)
    {
      sprintf(line, "%d\n",int1 + int2);
      n = strlen(line);
      if(write(STDOUT_FILENO,line,n) != n)
      {
        err_sys("write error");
      }

    }
    else
    {
      if(write(STDOUT_FILENO,"invalid args\n",13) != 13)
      {
        err_sys("write error");
      }
    }
  }
  exit(0);
}
```
对此程序进行编译，并保存为可执行文件add

下面的程序创建两个管道： 一个是协同进程的标准输入，另一个是协同进程的标准输出。它先从其标准输入读取两个数之后调用add协同进程，并将协同进程送来的值写到其标准输出。
```cpp
#include "apue.h"
static void sig_pipe(int);

int main(void)
{
  int n,fd1[2],fd2[2];
  pid_t pid;
  char line[MAXLINE];

  if(signal(SIGPIPE,sig_pipe) == SIG_ERR)
  {
    err_sys("signal error");
  }
  if(pipe(fd1) < 0 || pipe(fd2) < 0)
  {
    err_sys("pipe error");
  }
  if((pid = fork()) <0)
  {
    err_sys("fork error");
  }
  else if(pid > 0)
  {
    close(fd1[0]);
    close(fd2[1]);
    while(fgets(line,MAXLINE,stdin) != NULL)
    {
      n = strlen(line);
      if(write(fd1[1],line,n) != n)
      {
        err_sys("write error to pipe");
      }
      if((n = read(fd2[0],line,MAXLINE)) < 0)
      {
        err_sys("read error from pipe");
      }
      if(n == 0)
      {
        err_msg("child closed pipe");
        break;
      }
      line[n] = 0;
      if(fputs(line,stdout) == EOF)
      {
        err_sys("fputs error");
      }
      break;
    }
    if(ferror(stdin))
    {
      err_sys("fgets error on stdin");
    }
    exit(0);
  }
  else
  {
    close(fd1[1]);
    close(fd2[0]);
    if(fd1[0] != STDIN_FILENO)
    {
      if(dup2(fd1[0],STDIN_FILENO) != STDIN_FILENO)
      {
        err_sys("dup2 error to stdin");
      }
      close(fd1[0]);
    }
    if(fd2[1] != STDOUT_FILENO)
    {
      if(dup2(fd2[1],STDOUT_FILENO) != STDOUT_FILENO)
      {
        err_sys("dup2 error to stdout");
      }
      close(fd2[1]);
    }
    if(execl("./add","add",(char *)0) < 0)
    {
      err_sys("execl error");
    }
  }
  exit(0);
}
static void sig_pipe(int signo)
{
  printf("SIGPIPE caught\n");
  exit(1);
}
```
编译运行此程序，它会按预期工作。但是如果在它等待输入的时候杀死add协同进程，然后又输入两个数，那么程序对没有读进程的管道进行写操作时，会产生SIGPIPE信号。
```bash
1 2
SIGPIPE caught
```
# 死锁
&#160; &#160; &#160; &#160;我们用这个程序替换原来的add协同程序，则会发生死锁的问题：
```cpp
#include "apue.h"

int main(void)
{
  int int1,int2;
  char line[MAXLINE];
  while(fgets(line,MAXLINE,stdin) != NULL)
  {
    if(sscanf(line,"%d%d",&int1,&int2) == 2)
    {
      if(printf("%d\n",int1 + int2) == EOF)
      {
        err_sys("printf error");
      }
    }
    else
    {
      if(printf("invalid args\n") == EOF)
      {
        err_sys("printf error");
      }
    }
  }
  exit(0);
}
```
# 分析
&#160; &#160; &#160; &#160;我们第一个add程序是直接使用write和read的系统调用，后一个add程序则使用了标准I/O。因为标准输入现在变换成了管道，所以标准I/O的缓冲方式从行缓冲变成了全缓冲，标准输出也是如此，当子进程从其标准输入读取而发生阻塞时，父进程从管道读时也发生阻塞，于是产生了死锁。

# 解决
&#160; &#160; &#160; &#160;知道了原因，我们就可以通过设置标准I/O缓冲方式为行缓冲来解决问题
```cpp
if(setvbuf(stdin,NULL,_IOLBF,0) != 0)
{
    err_sys("setvbuf error");
}
if(setvbuf(stdout,NULL,_IOLBF,0) != 0)
{
    err_sys("setvbuf error");
}
```
其中：
- _IOFBF: 全缓冲
- _IOLBF: 行缓冲
- _IONBF: 无缓冲

重新编译并运行程序，发现死锁问题被解决了。
（全文完）