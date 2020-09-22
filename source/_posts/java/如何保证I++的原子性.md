---
title: 如何在多线程中保证i++的原子性
abbrlink: bc773bb8
date: 2020-09-19 14:32:32
tags:
---
> i++是不是一个原子操作？

相信很多人都知道，它并不是原子操作，为什么要写这篇文章，这也是我看了java并发编程中的第二章原子性相关的内容，重新思考的一个问题，如何保证i++的原子性？既然是总结，那我就从头开始理一下，围绕四个问题：1.什么是原子性操作;2.i++是不是原子操作;3.它为什么不是原子操作;4.如何保证i++的原子性;
<!-- more-->
# 剧透
文章可能有点长（啰嗦），这里先给结论（也希望大家可以看完，在火车上写了好几个小时🤦‍♂️）：

在多线程中，保证i++的原子性：
1. 最简单暴力的办法，通过java锁，`synchronized`或者`lock`接口的子类，给对象加锁，保证同时只有一个线程在操作
2. 通过java并发包下的一个原子操作的工具包，AtomicInteger.compareAndSet()方法实现cas(compare and swap,先比较在替换)，保证处理器在处理i++的时候，是原子操作，同时给变量加上volatile修饰符，保证线程的可见性。
3. 做法类似方法2，不过用getAndIncrement()代替cas赋值操作.

# 什么是原子性
 原子（Atomic），本意是“不能被继续分割的最小粒子”。原子性在我们计算机中一般表示为**原子操作，即不可被中断的一个或一系列操作**，这个概念在我们的编程语言中，数据库中，和操作系统及处理器中都有出现过。
# i++是不是原子操作
这个问题的答案很多人都知道，但是我们还是要用事实来说话，看下面代码

``` java
public class AtomicTest {

    public static int count = 0;

    public static void main(String[] args) {
        List<Thread> threadList = new ArrayList<>();
        for (int i = 0; i < 5000; i++) {
            threadList.add(new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    count++;
                }
            }));
        }
        for (Thread thread : threadList) {
            thread.start();
        }
        System.out.println("count：" + count);
    }
}
```
这个程序的很简单，创建50000个线程，执行相同的任务，每个线程执行1000次i++操作，我们期望的结果是50000*1000=50000000；，但是我执行了三次，结果分别为：
``` absh
count: 49764627
count: 49729044
count: 49584608
```
大家可以看到，结果并非我们期望的那样，三次结果各不相同，

为什么

# 多线程的i++为什么不是原子操作？
## i++在处理器中是如何执行的
i++虽然在java中是一条命令，但是它在处理器处理的时候，其实是三条命令:
``` bash
i = 1; #第一步先从内存中取出i的值
i + 1 = 1 + 1 = 2; #第二步计算 i + 1结果
i = 2; #第三步把结果赋值给i
```

所以如果两个线程同时做i++操作就可能发生下面的情况

1. 线程一获取i的值 i=1;
2. 线程二获取i的值 i=1;
3. 线程一计算i+1结果： i+1=1+1=2; 
4. 线程一把结果赋值给i：i+1=1+1=2; 
5. 线程二计算i+1结果： i+1=1+1=2; 
6. 线程二把结果赋值给i：i+1=1+1=2; 

最终就会导致两个线程执行完，i的值等于2,解决这个问题，就需要借助我们的并发包下面的原子操作类了，在java中，有很多原子操作类，它们都在 `java.util.concurrent.atomic` 这个包下面，比如：`AtomicBoolean.java`,`AtomicInteger.java`,`AtomicLong.java`等等。
我们来试一下用`AtomicBoolean.java`来重构一下我们的程序，看看结果如何
``` java
public class AtomicTest {

    public static volatile AtomicInteger count = new AtomicInteger(0);

    public static void main(String[] args) {
        List<Thread> threadList = new ArrayList<>();
        for (int i = 0; i < 5000; i++) {
            threadList.add(new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    count.getAndIncrement();
                }
            }));
        }
        for (Thread thread : threadList) {
            thread.start();
        }
        System.out.println("atomicCount" + count.get());
    }
}
```
同样执行三次，看一下结果：
```
atomicCount: 49976284
atomicCount: 49989257
atomicCount: 49976830

```
哪这个结果和上面的结果对比一下，大家会发现，咦？结果离我们预期的稍微有点接近哎。那是因为我们解决了i++在处理器中的原子操作问题，然后看下一个问题
## 多线程的变量可见性问题

这里需要先介绍一个java关键字，`volatile`,volatile是一个轻量级的synchronized,它的作用是在多线程编程中，保证**变量的可见性**，变量可见性就是说当一个线程修改了某个变量，其他线程并不一定会及时的取到变量值，因为每个线程都有自己本地的变量池。关于为啥在多线程编程中会有变量可见性问题，以及`volatile`和`synchronized`的原理, 如果有人感兴趣的话，我可以在新的文章里写，这里就不多说了。

那么我们用`volatile`来重写一下我们的程序

``` java
public class AtomicTest {

    public static volatile AtomicInteger count = new AtomicInteger(0);

    public static void main(String[] args) {
        List<Thread> threadList = new ArrayList<>();
        for (int i = 0; i < 5000; i++) {
            threadList.add(new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    count.getAndIncrement();
                }
            }));
        }
        for (Thread thread : threadList) {
            thread.start();
        }
        System.out.println("atomicCount" + count.get());
    }
}
```