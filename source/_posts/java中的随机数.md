title: java中的随机数
date: 2017-02-20 13:56:46
categories: java
tags:
  - java
------
# 引子
首先来看下面一段代码
```java
public class Program {
    public static void main(String[] args) throws IOException {
        int count = 100000;
        int limit = 100;
        Map<Integer, Integer> ticks = new HashMap<Integer, Integer>();
        Random random = new Random();
        while (count-- > 0) {
            int result = random.nextInt(limit);
            if (!ticks.containsKey(result)) {
                ticks.put(result, 0);
            }
            ticks.put(result, ticks.get(result) + 1);
        }
        for (int i = 0; i < limit; i++) {
            System.out.println(i + ":" + ticks.get(i));
        }
    }
}
```
这段代码的运行结果如下
```bash
0:1048
1:988
2:1015
3:955
4:1017
5:1018
6:990
7:1052
8:993
9:1012
10:1027
11:1008
12:976
13:994
14:990
15:994
16:1019
17:1072
18:1043
19:1015
20:1011
21:977
22:998
23:1000
24:1031
25:995
26:1008
27:1007
28:1001
29:998
30:1101
31:970
32:1008
33:969
34:931
35:1030
36:1023
37:994
38:1019
39:972
40:999
41:1002
42:929
43:948
44:1002
45:985
46:970
47:971
48:960
49:1019
50:1003
51:1027
52:1011
53:930
54:1004
55:1002
56:985
57:987
58:976
59:1037
60:969
61:1006
62:965
63:967
64:1047
65:1019
66:1009
67:973
68:998
69:959
70:999
71:984
72:964
73:1015
74:979
75:989
76:973
77:997
78:962
79:1043
80:1001
81:1010
82:1029
83:992
84:1049
85:994
86:975
87:1076
88:992
89:1009
90:977
91:1020
92:994
93:976
94:983
95:1028
96:1019
97:1062
98:936
99:1045
```
可以看到实际的结果在1000左右摆动。也就是说这样的代码产生的结果均匀分布。（笔者对上述代码进行了多次测试，结果都和这个是差不多的）均匀分布对于一个抽奖系统来说是非常重要的。例如，你花五块钱买一张彩票，我也花五块钱买一张彩票，大家抽中五万块钱的概率都是万分之一，
那么这个抽奖系统对于大家来说就是公平的。虽然这一段代码看起来是一段公平的代码，但事实上真的是这样子的吗？要理解其中的缘由，我们需要补充一些基础知识。

# “真”随机数
要生成一个“真”随机数，电脑会检测电脑外部发生的某种物理现象。比如说，电脑可以测量某个原子的放射性衰变。根据量子理论，原子衰变是随机而不可测的，所以这就是宇宙中的“纯粹”随机性。攻击者永远无法预测原子衰变的发生时间，也就不可能猜出随机值。
举个更实际的例子，电脑会根据环境中的噪音或者采取你敲击键盘的精确时间作为随机数据或熵的生成依据。举个例子，你的电脑监测到你某天下午2点以后敲击键盘的精确时间是0.23423523秒，有足够的这些特定长数字你就能得到一个熵源，也就可以生成“真”随机数。由于人不是机器，所以攻击者无法掌握你的敲击时间。
Linux中的/dev/random随机设备生成随机数，“阻拦”访问直到熵积累量足够才返回一个真随机数。（熵，热力学中表征物质状态的参量之一，用符号S表示，其物理意义是体系混乱程度的度量。）（注：/dev/random产生随机数的效率十分低下，很难运用到生产环境中）

# 伪随机数
伪随机数这个概念是相对于“真”随机数而言。电脑通过发送种子数值，运用算法产生某个看起来像随机数的数字，但是实际上这个数字是可以预测的。因为电脑没有从环境中收集到任何随机信息。

# 如何判断一个随机数发生器的优劣

德国联邦信息安全办公室给出了随机数发生器质量评判的四个标准

- K1——相同序列的概率非常低
- K2——符合统计学的平均性，比如所有数字出现概率应该相同，卡方检验应该能通过，超长游程长度概略应该非常小，自相关应该只有一个尖峰，任何长度的同一数字之后别的数字出现概率应该仍然是相等的等等
- K3——不应该能够从一段序列猜测出随机数发生器的工作状态或者下一个随机数
- K4——不应该从随机数发生器的状态能猜测出随机数发生器以前的工作状态

文章开头那段代码，只满足了K2这个要求。其实K1，K3,和k4一个都不符合。

# 分析
笔者好奇的是，这段代码在C#中是不可能均匀分布的。因为在C#中new Random()是以1970年1月1日到当前时间的毫秒数作为线性同余算法的种子的。而在现代计算机中1毫秒内可以运行几十万次的while循环，因此你会发现获得的随机数大都相同。
来看下面的实验
```C#
Bitmap bmp = new Bitmap(300, 300);
Graphics g = Graphics.FromImage(bmp);
SolidBrush b = new SolidBrush(Color.Black);

Random random = new Random();
int count = 100000;
while (count-- > 0)
{
    int x = random.Next(300);
    int y = random.Next(300);
    g.FillRectangle(b, x, y, 1, 1);
}
```
这段程序生成的图片如下
![](http://7xlovv.com1.z0.glb.clouddn.com/hello.png)

将Random放入While循环中
```C#
Bitmap bmp = new Bitmap(300, 300);
Graphics g = Graphics.FromImage(bmp);
SolidBrush b = new SolidBrush(Color.Black);


int count = 100000;
while (count-- > 0)
{
    Random random = new Random();   
    int x = random.Next(300);
    int y = random.Next(300);
    g.FillRectangle(b, x, y, 1, 1);
}
```
![](http://7xlovv.com1.z0.glb.clouddn.com/Csharp2.png)

这里只能看到零星的几个黑点，可以看到两者的差异非常大。但是在Java中这两种写法产生的结果是一致的。（实际上在早期的JDK版本也是和C#同样的结果）

那么这到底是怎么回事呢？

我们来看一下Java中Random类的构建函数
```java
  public Random() {
        this(seedUniquifier() ^ System.nanoTime());
    }
```

再来看一下seedUniquifier()这个函数
```java
private static final AtomicLong seedUniquifier
        = new AtomicLong(8682522807148012L);

private static long seedUniquifier() {
        // L'Ecuyer, "Tables of Linear Congruential Generators of
        // Different Sizes and Good Lattice Structure", 1999
        for (;;) {
            long current = seedUniquifier.get();
            long next = current * 181783497276652981L;
            if (seedUniquifier.compareAndSet(current, next))
                return next;
        }
    }
```

可以看到这就是一个线性同余的算法，其中的种子是一个64位整型：8682522807148012L

我们截取这一段代码运行一下：
```java
int i = 10;
while (i-- > 0) {
    System.out.println(seedUniquifier());
}
```
结果如下：
```bash
8006678197202707420
-3282039941672302964
3620162808252824828
199880078823418412
-358888042979226340
-3027244073376649012
2753936029964524604
-9114341766410567060
-4556895898465471908
7145509263664170764
```
也就是说无论你运行多少次都是这个结果。那么再来看一下System.nanoTime()这个函数

在代码的注释中可以找到
```bash
/**
     * Returns the current value of the running Java Virtual Machine's
     * high-resolution time source, in nanoseconds.
```
这句话的意思是返回当前java虚拟机的当前时间，是一个高精度的时间源，单位为纳秒。

# 抽奖程序

那么这时候我们来分析一下在java中每次都new一个 Random实例时，如何破解这样一个抽奖程序。

- 首先我们要知道我们抽奖的序号，也就是在那么多抽奖先后顺序，通过这个我们可以得出线性同余的随机数，因此，我们将抽奖序号设为N

- 抽奖的时间点，也就是System.nanoTime(),我们设为T

- 奖池

其中，如果作为一名抽奖程序的开发人员要知道第一个和第三个条件其实很简单。至于System.nanoTime()要获取可以说是难于登天。所以，这段程序从表面上看似乎并没有什么漏洞。

如果我们将时间放慢一亿倍（你能接近光速么？），我们就能精确的控制在哪一秒点下去就能中五万元了不是么？不过要接近光速似乎是不可能的事情。

不过，我们换一种方式来分析问题。虽然纳秒太细我们无法精确的控制，但是我们可以计算出在哪一秒内出现中五万元的纳秒数最多，于是我们可以选择在那一秒去点击抽奖按钮。这样我们中五万元的概率是不是就比别人高了许多？


（全文完）

