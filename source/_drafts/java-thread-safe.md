---
layout: drafts
title: Java中的线程安全和一些常见的保证线程安全的方法
tags:
  - Java
  - 多线程
  - 线程安全
categories:
  - Java
date: 2019-02-26 11:50:00
---


# 什么是线程安全
## 并发编程／多线程
在讲Java的线程安全之前，首先我们要知道，Java的多线程编程，可以简单的理解为，让程序同时干多件事情，比较经典一点的就是，在UI程序中，一般会有一个主线程来负责渲染UI界面（比如安卓中的main线程），这个线程一般都是一个死循环，不断的渲染界面的同时，也不让程序退出。
在这个时候，如果我们想要发送一个网路请求怎嚒办，我们知道一个网络请求，可能会需要几秒乃至十几秒，如果我们直接在渲染UI的界面干这个事情，就会卡死线程，就没有办法渲染UI了，具体在安卓程序上的表现就是卡顿或者是ANR（应用程序无响应）

因此，很多时候，我们就需要使用多线程，让他们同时执行。

## 线程安全
上面讲到，有的时候，我们需要使用多线程，这就会带来一个新的问题，就是如果2个线程同时访问一个实例、变量的问题。
为了说明问题 ，我这里做了一个多线程问题验证的框架。
```
public abstract class IThreadSafeTest {
    //目标数值
    public static final int END_VALUE = 0xffffff;

    public abstract void add(); //在这里实现线程安全的加法

    public abstract boolean verify();//验证成功调用add()次数
}
```
测试的时候很简单，实现上面的接口，然后用下面的方法验证
```
public void test(Class<? extends IThreadSafeTest> cls) {//cls 是上面的一个实现类
    IThreadSafeTest iThreadSafeTest = cls.newInstance();
    int threadCount = IThreadSafeTest.END_VALUE / THREAD_NUM;//线程的数量
    Thread[] threads = new Thread[THREAD_NUM];
    for (int i = 0; i < THREAD_NUM; i++) {
        threads[i] = new Thread(new AddRunAble(iThreadSafeTest, threadCount));
        threads[i].start();
    }
    for (int i = 0; i < THREAD_NUM; i++) {
        threads[i].join();
    }
    assert iThreadSafeTest.verify();//判断累加的结果是否正确，如果错误，就表示线程一定不安全，如果正确，大概率是对的，可以多跑几次。
}
```

然后我们来做一个完全不做线程安全的实现，这里就只贴出add方法了：
```
public void add() {
    value = value + 1;
}
```
运行结果如下：
{% asset_img run-1.png 线程安全问题 %}

因此，我们用到多线程的时候，就有可能有线程安全的问题。本文就主要介绍一下几种处理线程安全的方法。

# 什么时候会有线程安全的问题
在讲线程安全之前，需要简单的介绍一下Java的内存模型。再将内存模型之前，再简单介绍一下内存结构（感觉越扯越远了，但是是为了说明清楚问题）。
Java的内存结构，也就是运行时的数据区域，包括：方法区、Java栈、堆、本地方法栈、程序计数器，借用一张图来表达：
{% asset_img mms.png Java内存结构 %}
> 图来自博客：https://www.cnblogs.com/lewis0077/p/5143268.html
## Java内存结构
### 程序计数器
下面重点说一下和多线程有关的吧，程序计数器，主要保存当前正在执行的程序的内存地址，这个就和我们要说的多线程有关心了，大家都知道，cpu一个时间分片，只能执行一个程序，他的线程同时执行，是在不同线程之间来回切换，切换的时间足够快， 就让我们感觉他们是同时运行的，所以一个线程的执行，并不是线性的，当多个线程交叉执行的时候，中断当前程序，就需要保存当前执行到哪儿了，以及当前线程的一些数据，方便下次再执行的时候，进行恢复。
每一个线程都需要一个独立的程序计数器，因为是独立的，所以他们不会相互的影响，是 线程私有 的，所以，这一块也是线程安全的。
### Java栈
栈和堆，应该是Javaer最熟悉的2块内存了，Java的栈，都是和线程关联在一起的，开一个新线程，就会有一个新的对应的栈，一个栈当中，又包括了多个栈帧，每调用一个方法，就会入栈一个栈帧，记录这一次的方法调用信息，每返回一个栈，就会把最上面的栈帧给出栈了。
所以，当我们无限制的在方法里面调用方法（比如：递归），因为栈的空间是有限的，当栈满了一个，再继续调用方法，就会给出：StackOverflowError 错误了。
通过上面的描述，我们大概知道了，一个线程就对应一个栈，这样每个线程的栈也是独立的，所以，栈这一块也是线程安全的，不会因为多个线程同时运行而出现线程安全问题。
### 堆(Heap)
堆是我们所使用的内存中，最大的一块，也是被所有Java线程所共享的，关于堆，说的最多的是GC，我们今天主要要说的不是GC，是线程共享。
一旦数据被多个线程共享了，那就存在安全的问题了。但是为啥会有这里共享就会有线程安全问题呢？这就要介绍一下签名提到的Java的内存模型了。
## Java内存模型
Java的内存模型，简单点的理解 ，就是定义了程序中的各个变量的存取规则。但是需要说明一点的是，这里的变量是指能够被线程共享的变量（实例的成员变量、静态变量、数组对象等，不包括方法参数，局部变量）。
Java内存模型，规定了变量存储在主内存中(Main memory),但是线程不直接在主内存存取数据，每个线程又自己的工作内存（Working Memory），线程要访问一个变量之前，需要先从主内存拷贝到工作内存，然后再在自己的工作内存里面玩。（这里顺带可以了解一下volatile 关键字），现在只能访问自己工作内存中的数据，不能访问别的线程的工作内存，所以要交换数据，必须通过主内存。大概就是下面这样的：
{% asset_img thread-working-memory.png Java内存模型 %}
> 图来自博客：https://www.cnblogs.com/lewis0077/p/5143268.html

这样，线程1要和线程2交换数据，就要这样：
- 线程1把工作内存中的数据，刷新到主内存中；
- 线程2把主内存中的数据拷贝到线程2的工作内存，线程使用更新以后，再刷新回主内存中。

这样的话，问题就来了，假如：内存中有个变量是A=1，线程1做-1处理，线程2做+1处理。正确的来说，运算完了以后 ，内存中的变量还是A=1，但是如果遇到下面这样的顺序：
- 线程1从主内存读取A=1到线程1的工作内存，做-1运算，A=0；
- 线程2从主内存读取A=1到线程1的工作内存，做+1运算，A=2；
- 线程1把工作内存中变量A=0，刷新到主内存；
- 线程2把工作内存中变量A=2，刷新到主内存；

搞完了以后，发现本应该是A=1的数据变成了A=2，这个就是多线程的内存共享带来的问题，下面我们 用一个时序图来表示：

{% asset_img run-order.png Java多线程问题内存时序图 %}
## 存在线程安全问题的场景
前面比较仔细的解释了，为啥会有线程安全问题，怎嚒出现的，我们稍微做一个总结，其实也就是**存在线程共享内存的地方，就有可能出现线程安全问题**，具体的来说，就是：
- 多个线程访问同一个实例的成员变量；
- 多个线程访问类静态变量；
- 多个线程访问数组对象；
- 暂时想到的就这个几个，反正就是多个线程访问同一内存的场景。

# 实现线程安全的方式
既然Java多线程并发编程，存在这个问题，那么肯定要有解决办法，这里介绍一下，常见的几种实现线程安全的方式。
## synchronized关键字
在说明synchronized关键字之前，我们先要理解一个概念：互斥锁，顾名思义，就是相互排斥，同一时间只能有一个访问到目标对象的锁。
简单点说 ，用synchronized修饰的代码，同一时间，只能有一个线程能够访问，其他线程必须等到这个线程离开synchronized修饰的代码，才能够再次进入并且访问这里面的代码，当时，synchronized的使用也有很多种方法，这里我简单的这个总结：
{% asset_img synchronized-use.png synchronized的使用场景 %}
> 图片来源：https://www.jianshu.com/p/d53bf830fa09
这一块我都写了相关的实现，使用了开头提到的多线程验证的框架，代码在：

## ReentrantLocak（重入锁）
synchronized虽然可以解决多线程的问题了，但是遇到一些负责的问题，可能就不够用了，比如：
- 我们要判断某段代码当前可否访问，不能访问就不妨问了，直接掉过做其他事情；
- 比如我们等待锁的时候，设置一个等待的时常，超过一定时间还等不到锁，就放弃；
- 写操作加锁，但是读的时候 ，是不需要加锁的等等。
遇到上面的这些问题，我们就可以使用ReentrantLocak来解决。
他为啥叫重入锁呢？一个线程可以多次获取并且进入。
### 基本用法
他的基本使用差不多是这样：
```
public void add() {
    mReentrantLock.lock();//加锁，不让其他线程访问
    value++;
    mReentrantLock.unlock();//释放，其他线程就可以访问了
}
```
**这里需要注意，lock了几次，就需要unlock几次**
### Lock的公平锁和非公平锁
```
Lock lock=new ReentrantLock(true);//公平锁
Lock lock=new ReentrantLock(false);//非公平锁
```
公平锁，就是按照线程后去锁的顺序排队，按照顺序来获取锁。非公平锁就是抢占，谁先获取锁，全靠本事，先到不一定先得。
### 相关的一些使用方法
- isLock() 是否有任意线程获取到了这个锁。 todo
- getHoldCount() 查询当前线程保持此锁的次数，也就是执行此线程执行lock方法的次数
- getQueueLength（）返回正等待获取此锁的线程估计数，就是调用了lock方法，正在等待的线程数
- hasQueuedThreads()是否有线程等待此锁
- hasQueuedThread(Thread thread)查询给定线程是否等待获取此锁
- isFair() 该锁是否公平锁
- isHeldByCurrentThread() 当前线程是否获取到了锁
- lockInterruptibly（）获取锁，直到获取成功或者线程被中断
- tryLock（）获取锁，获取成功返回true，获取失败立即返回，不会等待
- tryLock(long timeout,TimeUnit unit) 获取锁，获取成功返回true，如果当前锁被占用了，会等timeout。

### await/signal 方法
当一个线程A获得了A锁，运行中又需要B锁，但是这个时候B锁被线程B占用了，这个时候线程A就只有等待获取B锁成功，才可以继续。
那嚒我们想想，如果还有其他的线程需要A锁呢？就会造成一个链式的等待，降低运行效率。
首先我们通过锁创建一个Condition
```
Condition sCondition = sReentrantLock.newCondition();
```
所以，我们获取到了一个锁以后，可以主动放弃运行，再等条件运行的时候，把它唤醒。
```
sReentrantLock.lock();
try {
    sCondition.await();//等待，并且释放获取到的锁
} catch (InterruptedException e) {
    e.printStackTrace();
}
sReentrantLock.unlock();
```
上面这样就可以暂停线程，并且释放他获取的A锁，这样我们在其他线程，就可以获取A锁并且继续运行，运行完了以后，通过signal()方法，就可以再次唤醒线程A。
```
sReentrantLock.lock();
sCondition.signal();//唤醒自线程
sReentrantLock.unlock();//释放主线程的锁，不然，线程A获取不到锁,还是会阻塞
```
这样有没有觉得非常强大，可以达到资源的最大利用。但是这里需要注意：一个condition对象的signal方法和该对象的await方法是一一对应的。

### tryLock和lock和lockInterruptibly的区别
- lock，就是直接获取锁，获取不到就一直等待
- tryLock，尝试获取锁，无论是否成功，都立即返回，成功返回true，失败返回false。另外，还可以tryLock(long timeout,TimeUnit unit),给一个最大等待时间，获取timeout时间还获取不到锁，就返回false
- lockInterruptibly 如果获取锁的过程中，线程被中断，lock不会抛异常（不中断，会继续运行），lockInterruptibly会抛异常。







# 总结
- 

# 学习源码
纸上谈来终觉浅，绝知此事要躬行。
文中提到的源代码，你都可以参考：
[https://github.com/zjiecode/learn-java/tree/feature/feature/thread-safe](https://github.com/zjiecode/learn-java/tree/feature/thread-safe)

# 参考资料
> https://www.cnblogs.com/lewis0077/p/5143268.html
> https://www.cnblogs.com/-new/p/7256297.html
> 



