title: 多线程一定比单线程快吗
date: 2017-08-10 15:27:07
categories: performance
tags:
  - performance
  - cpu
  - thread
------
## 问题
首先我们来看一下需求：
> 有四个整型数（初始化为0）存在我们计算机的内存之中，现在要求对每个数分别进行++操作，重复这个动作1000000000次。

这很显然是一个计算密集型的过程，其间并不会涉及任何的io操作，因此，我们可以考虑用四个线程来同时对这四个数进行操作（用四个线程是因为笔者的电脑CPU物理核数为4）,这样可以最大程度地利用我们的cpu资源。

## 实现
接下来，我们用程序来实现这个需求，笔者用的语言是C#，这个实验和使用的语言关系不是很大，各个语言之间的差异很小。
```csharp
class Program
{
    private static int m_numOfProcessor = 4;
    private static int m_step = 1;
    private volatile static int[] m_array = new int[m_numOfProcessor * m_step];
    private static readonly int m_ticks = 1000000000;
    static void Main(string[] args)
    {
        var events = InitEvents(m_numOfProcessor);
        Stopwatch watch = new Stopwatch();
        watch.Start();
        for (int i = 0; i < m_numOfProcessor; i++)
        {
            new Thread(new ParameterizedThreadStart((o) =>
            {
                UpdateCounter(Convert.ToInt32(o) * m_step);
                events[Convert.ToInt32(o)].Set();
            })).Start(i);
        }
        WaitHandle.WaitAll(events);
        watch.Stop();
        Console.WriteLine(String.Format("step {0} elapsed:" + watch.ElapsedMilliseconds, m_step));
        Print(m_array, m_step);
        Console.Read();
    }
    private static void Print(int[] array,int step)
    {
        for (int i = 0; i < array.Length; i = i + step)
        {
            Console.WriteLine(array[i]);
        }
    }
    private static ManualResetEvent[] InitEvents(int num)
    {
        ManualResetEvent[] events = new ManualResetEvent[4];
        for (int i = 0; i < num; i++)
        {
            events[i] = new ManualResetEvent(false);
        }
        return events;
    }
    private static void UpdateCounter(int position)
    {
        for (int i = 0; i < m_ticks; i++)
        {
            m_array[position] = m_array[position] + 1;
        }
    }
}
```
在这个示例中，我们将这四个整数放在一个数组中，当m_step为1时，这四个数在内存中是连续的。这里我们开启了四个线程同时行进计算，得到的结果如下：
```bash
step 1 elapsed:15495
1000000000
1000000000
1000000000
1000000000
```

我们一开始的想法是通过多线程来充分利用CPU的计算资源，那么，我们现在来验证多线程是否确实比单线程速度快：
```csharp
for (int i = 0; i < m_numOfProcessor; i++)
{
    //new Thread(new ParameterizedThreadStart((o) =>
    //{
    //    UpdateCounter(Convert.ToInt32(o) * m_step);
    //    events[Convert.ToInt32(o)].Set();
    //})).Start(i);
    UpdateCounter(i * m_step);
}
//WaitHandle.WaitAll(events);
```
调整程序为单线程执行，结果如下：
```bash
step 1 elapsed:7741
1000000000
1000000000
1000000000
1000000000
```

结果竟然是单线程比多线程的速度快！

接下来，我们调整一下m_step的值分别为2、4、8、16，来看一下不同的步长在多线程和单线程模式下的各自的性能表现：

| m_step | 多线程 | 单线程 |   
|----------|----------|----------|
|  1 |  15495 | 7741  |
|  2 |  15768 | 7760  |
|  4 |  15485 | 7667  |
|  8 |  **11433** | 7752  |
|  16 |  **5797** | 7784  |

这里，我们发现当步长为8和16的时候，多线程性能突然提升，并且在16的时候超过了单线程的性能，而这一切又是为什么呢？

## 分析
要解释这个结果，我们要先理解CPU的工作原理。
1) cpu从来都不直接访问主存, 都是通过cpu cache间接访问主存。
2) 每次需要访问主存时, 遍历一遍全部cache line, 查找主存的地址是否在某个cache line中。
3) 如果cache中没有找到, 则分配一个新的cache entry, 把主存的内存copy到cache line中, 再从cache line中读取。

那什么又是cache line呢？
现代的cpu从主存读取数据并不是一个字节一个字节读取，而是一整块一整块地读取，那么究竟一次会读取多少数据呢，这就取决于CPU的cache line的大小。CPU将它的cache划分成一块一块的，一块这样的存储区域就是一个cache line。
在现代计算机CPU的cache line大小一般为32Byte或64Byte，我们要如何查看cache line的大小呢？可以使用 [CoreInfo](https://docs.microsoft.com/zh-cn/sysinternals/downloads/coreinfo)工具。
```bash
Logical Processor to Cache Map:
**------  Data Cache          0, Level 1,   32 KB, Assoc   8, LineSize  64
**------  Instruction Cache   0, Level 1,   32 KB, Assoc   8, LineSize  64
**------  Unified Cache       0, Level 2,  256 KB, Assoc   8, LineSize  64
********  Unified Cache       1, Level 3,    8 MB, Assoc  16, LineSize  64
--**----  Data Cache          1, Level 1,   32 KB, Assoc   8, LineSize  64
--**----  Instruction Cache   1, Level 1,   32 KB, Assoc   8, LineSize  64
--**----  Unified Cache       2, Level 2,  256 KB, Assoc   8, LineSize  64
----**--  Data Cache          2, Level 1,   32 KB, Assoc   8, LineSize  64
----**--  Instruction Cache   2, Level 1,   32 KB, Assoc   8, LineSize  64
----**--  Unified Cache       3, Level 2,  256 KB, Assoc   8, LineSize  64
------**  Data Cache          3, Level 1,   32 KB, Assoc   8, LineSize  64
------**  Instruction Cache   3, Level 1,   32 KB, Assoc   8, LineSize  64
------**  Unified Cache       4, Level 2,  256 KB, Assoc   8, LineSize  64
```
在笔者的计算机上看到的L1的cache line大小为64。这里显示我的L1的数据缓存为32KB，指令缓存为32KB。这里要注意的是，L1 cache是处理器独享，L2 cache是成对处理器共享的。
也就是说，笔者的四个线程，每个线程可以享用32KB的一级缓存。在上面的实验中，正是因为独享的一级缓存导致的程序性能低下。
在这里，我们可以联想一下我们宏观架构中的分布式缓存系统，如果我们将同一份数据存到多个地方的时候，那么数据一致性的维护将是一个非常头痛的问题。当有一份数据发生了变化，我们就必须通过某种手段来保证其它地方的数据也被更新。为了数据的一致性，势必会让我们损失很多性能。
我们回到上面的实验：，
一个32位整型数的大小为4Byte，而一个cache line长度为64Byte，因此，在一个cache line中，可以存放 64 / 4 = 16 个 32位整型数。
- 当步长为1时，四个数全部在同一个cache line中
- 当步长为2时，四个数全部在同一个cache line中
- 当步长为4时，四个数全部在同一个cache line中
- 当步长为8时，每两个处于不同的cache line中
- 当步长为16时，每个数都处于不同的cache line中

到这里我们就很好解释上述实验所显示的结果，正是因为多个CPU处理器同时操作了同一个cache line，一旦某一个处理器更新了cache line的数据，会导致其它处理器同一个cache line的数据失效，那么其它处理器就必须从主存再次读取，这正是导致其性能低下的原因。
（全文完）

