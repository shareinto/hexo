title: 浅谈java中List接口
date: 2016-12-15 14:18:11
categories: java
tags:
  - java
------
# List接口的定义
List接口在java中算是使用频率相当高的一个接口，我们先来看一下它的定义：
```java
public interface List<E> extends Collection<E>{
    ...
    boolean add(E e);
    int size();
    E get(int index);
    Iterator<E> iterator();
    ...
}
```
这里只列出一些比较重要的方法，相比其它语言里面的类似的接口，可以说是大同小异，它的设计并没有什么问题。
# 实现
在jdk中，关于这个接口有三个实现，分别是ArrayList,LinkedList和Vector,我们分别来看一下它们的定义：
```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
```java
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
```java
public class Vector<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

既然实现了List接口，那我们便可以用List指针来接收这三个类的实例。例如：
```java
List list = new ArrayList();
List list = new LinkedList();
List list = new Vector();
```
# 迭代
在平常使用List接口的过程中，我们经常需要遍历List里面的元素。
于是我们可以这样写代码：
```java
for (Interator inter = list.Interator();iter.hasNext();){
     Object obj = inter.next();
}
```
我们甚至还可以这样：
```java
for (Object obj : list) {
}
```
为什么可以这样？这是因为AbstractList实现了Iterable接口，而这里仅仅只是一种语法糖而已，实际代码编译后还是会被转成第一种写法。
（大家可以想想这里为什么要有一个Iterable接口，而不直接实现Iterator接口）

文章写到这里似乎并没有什么价值，但是我们发现在List接口中存在：
```java
E get(int index);
```
这样一个方法，我们似乎可以改改上面遍历的方式：
```java
for (int i = 0; i < list.size(); i++) {
     Object obj = list.get(i);
}
```
我相信大家在日常编码中会经常使用到这种方式。
# 随机访问
&#160; &#160; &#160; &#160;这种通过下标访问的方式，我们把它称之为RandomAccess。我们知道数组这种数据结构对这种随机访问的天生支持（事实上ArrayList和Vector就是用数组实现的），也就是说它的访问效率是非常高的。
现在我们回过头来看JDK中对List的三种实现，我们会发现其中的ArrayList和Vector竟然实现了一个叫RandomAccess接口，查看它的定义：
```java
public interface RandomAccess {
}
```
竟然是一个空接口？好吧这种接口的作用实际上是一种标记接口，对于它的使用，往往需要配合instanceof这种[RTTI](http://baike.baidu.com/link?url=c6vVFXT41_awqHe0TVcfrR74uwaprffqcyzQP4qw_o3VQQ0L2OSvQgzxWGR_a6_argI5qoOg2Pe5P_cv2X0YEq)的方式。可以说并不是一种很理想的方式。

# 性能
我们可以写一个小程序来测试一下使用迭代器和使用RandomAccess的性能差异
```java
public static void travelwithoutIterator(List list, int count) {
    long startTime;
    long endTime;
    startTime = System.currentTimeMillis();
    for (int a = 1; a <= count; a++) {
        for (int i = 0; i < list.size(); i++) {
            list.get(i);
        }
    }
    endTime = System.currentTimeMillis();
    long interval = endTime - startTime;
    System.out.println("不使用迭代器的间隔时间：" + interval);
}

public static void travelwithIterator(List list, int count) {
    long startTime;
    long endTime;
    startTime = System.currentTimeMillis();
    for (int a = 1; a <= count; a++) {
        for (Iterator iter = list.iterator(); iter.hasNext(); ) {
            iter.next();
        }
    }
    endTime = System.currentTimeMillis();
    long interval = endTime - startTime;
    System.out.println("使用迭代器的间隔时间：" + interval);
}
public static void addObject(List list, int n) {
    for (int m = 1; m <= n; m++) {
        list.add("" + m);
    }
}
```
在main中：
```java
int number = 100000;
int count = 100;
System.out.println("遍历ArrayList：");
addObject(list, number);
travelwithoutIterator(list, count);
travelwithIterator(list, count);
```
结果是:
```bash
遍历ArrayList：
不使用迭代器的间隔时间：5
使用迭代器的间隔时间：12
```
我们对于ArrayList，使用RandomAccess的效率要比使用迭代器高不少，这对于一些对于性能要求比较苛刻的程序来说，可能会是一个优化的点。

但是我们现在换成LinkedList来试一试：
```java
System.out.println("遍历LinkedList：");
addObject(list, number);
travelwithoutIterator(list, count);
travelwithIterator(list, count);
```
结果是：
```bash
遍历LinkedList：
不使用迭代器的间隔时间：1043247
使用迭代器的间隔时间：139
```
what the fuck! 这个坑也太大了吧！对于List的使用者来说，或者说正在编写一个框架的人，我们经常不知道List指针会接到什么样的一个具体实例，于是乎，我们只能以这种贴膏药的方式来修补我们的程序：
```java
if (list instanceof RandomAccess) {
    for (int i = list.size(); i < list.size(); i++) {
        Object obj = list.get(i);
    }
} else {
    for (Object obj : list) {
    }
}
```

# 设计的问题
&#160; &#160; &#160; &#160;到这里，我们可以看出LinkedList明显是不应该实现List这么大的一个接口了，或者说早期的jdk设计人员并没有考虑到这样的性能问题。我们再回过头来看RandomAccess接口，jdk设计人员似乎也是意识到了这一点，才搞出了这么一个东西。
那会为什么不让LinkedList直接去掉List接口呢？（在C#中LinkedList并没有实现IList接口）我想这应该是Java设计人员始终坚持的兼容性原则，这跟Jvm始终不愿意引入泛型是一个道理。


