---
title: Java多线程之可见性
date: 2016-08-02 09:31:16
tags: [java,multi-thread]
categories:
- Java
- thread
---

> Java多线程系列文章：
> https://zhum.in/blog/categories/Java/thread/

- - -

# 线程的创建方式
## 继承Thread类
```java
class MyThread extends Thread {
	@Override
	public void run() {
		// override run method
	}
}

// 线程的启动
MyThread myThread = new MyThread();
myThread.start();
```

## 实现Runnable接口
```java
class MyThread implements Runnable {
	public void run() {
		// override run method
	}
}

// 线程的启动
MyThread myThread = new MyThread();
Thread thread = new Thread(myThread);
thread.start();
```

<!-- more -->

## 两种方式的区别
显然继承Thread类的方式会受到单继承的限制。另外，使用实现Runnable方式的类初始化多个线程可以使多个线程共享资源，如：
```java
class MyThread implements Runnable {
	int sourceCount = 10;
		public void run() {
		// override run method
	}
}

MyThread myThread = new MyThread();
Thread thread1 = new Thread(myThread);
Thread thread2 = new Thread(myThread);
Thread thread3 = new Thread(myThread);

thread1.start();
thread2.start();
thread3.start();
```

# 线程的生命周期
1. 创建：当new一个线程对象的时候，就已经创建了一个线程。

2. 就绪状态：执行start()方法之后，线程就处于了就绪状态，这个时候线程其实已经具备了运行的条件，但是还未运行，被加入到线程队列当中等待CPU的资源调度。

3. 运行状态：当线程队列中的线程获取到了CPU的时间片，就被调度执行了，此时会进入到覆写的run()方法当中，执行业务逻辑操作。

4. 终止：当业务逻辑操作执行完毕后，线程也就被自动的销毁掉了。

5. 阻塞状态：一个正在执行的线程由于某些原因被终止掉了，这个时候就会让出CPU的资源，自动暂停当前的执行，这个时候就已经处于阻塞状态了。

 比如调用了sleep()方法，线程将被阻塞直到sleep()方法的timeout结束，阻塞解除，或者调用了wait()方法被唤醒，阻塞才被解除，线程重新回到就绪状态，等待CPU的资源调度。或者调用了wait()、suspend()方法或者join()方法。

# 守护线程
Java中的线程分为两类：
1. 用户线程：运行在前台，执行具体的任务，如程序的主线程，连接网络的子线程都是用户线程。

2. 守护线程：运行在后台，执行守护任务，为其他前台线程服务。
守护线程的特点是一旦所有用户线程都结束运行，守护线程会随着JVM一起结束工作。一般应用在数据库连接池中的监测线程、JVM虚拟机启动之后的监测线程等。最常见的守护线程就是JVM中的GC垃圾回收线程。

守护线程注意事项：
1. 调用`Thread.setDaemon()`方法即可设置守护线程，但是必须在start()方法之前调用
2. 守护线程中产生的新线程也是守护线程
3. 读写操作和计算逻辑不可设置为守护线程

# 可见性介绍
1. 可见性：一个线程对共享变量值的修改，能够及时被其他线程看到。

2. 共享变量：如果一个变量在多个线程的工作内存中都存在副本，那这个变量就是这几个线程的共享变量。

3. 线程的工作内存：Java内存抽象出来的概念。

4. Java内存模型（JMM-Java Memory Model）：描述了Java程序中各种变量（线程共享变量）的访问规则，以及在JVM中将变量存储到内存和从内存中读取出变量这样的底层细节。

5. 所有变量都存储在主内存中；而每个线程有自己独立的工作内存，在工作内存中的保存的该线程使用的变量（此变量是主内存中变量的副本）。

6. Java内存的规定：
 - 线程对共享变量的所有操作都必须在自己的工作内存中进行，不可直接从主内存中读写；
 - 不同线程之间无法直接访问其他线程工作内存中的变量，线程间的变量值的传递需要通过主内存

7. 共享变量可见性实现原理：
```bash
线程1 -> 工作内存1中变量X -> 更新到主内存中 -> 工作内存2中的变量X得到更新 -> 线程2
```

# 可见性的实现方式
## synchronized实现
`synchronized实现`关键字不仅可以实现线程的同步（原子性），也可以实现内存的可见性。

JMM中关于synchronized的两条规定：
1. 线程解锁前，必须把共享变量的最新值刷新到内存中
2. 线程加锁时，将清空工作内存中共享变量的值，从而使用共享变量时需要从主存中中重新读取最新的值。（注意：加锁与解锁需要是同一把锁）

也就是说，线程解锁前对共享变量的修改在下次加锁时对其他线程可见。

线程执行互斥代码的过程：
1. 获得互斥锁
2. 清空工作内存
3. 从主内存拷贝变量到最新副本到工作内存
4. 执行代码
5. 刷新变量到主内存
6. 释放互斥锁

## 指令重排序
指令重排序：代码书写的顺序与实际执行的顺序不同，指令重排序是编译器或处理器为了提高程序性能而做的优化
1. 编译器优化的重排序
2. 指令级并行重排序
3. 内存系统的重排序

Java语言在单线程中遵循`as-if-serial`语义，即不会将有数值依赖的语句进行重排，因此重排序不会给单线程带来内存可见性问题。但是多线程中程序交错执行时，重排序可能会造成内存可见性问题

导致共享变量不可见的三个原因：
1. 线程的交叉执行
2. 重排序结合线程交叉执行
3. 共享变量更新后的值没有在工作内存与主内存间及时更新

## synchronized实现可见性原理
1. 原子性：由于锁的关系，线程之间不允许交叉执行；相当于给该线程（或当前运行的有且仅有一个的线程）加了一把锁，外面的线程无法进入，更别提互相交叉执行。
2. 原子性`as-if-serial`语义：线程不能交叉执行，重排序对于单线程不能影响运行结果。
3. 可见性：共享变量的更新执行。

## synchronized与对象锁
1. 当两个并发线程访问同一个对象object中的这个synchronized（this）同步代码块时，一个时间内只能有一个线程得到执行；
2. 当一个线程访问object的一个synchronized（this）同步代码块时，另一个线程仍然可以访问该object中的非synchronized（this）同步代码块
3. 当一个线程访问object的一个synchronized（this）同步代码块时，他就获得了这个object的对象锁，结果其他线程对该object对象所有同步代码部分的访问都被暂时阻塞

# volatile关键字
1. 能够保证volatile变量的可见性。
2. 不能保证volatile变量复合操作的原子性。
3. 共享数据必须用private修饰

## volatile如何实现可见性
volatile关键字通过加入内存屏障和禁止重排序优化来实现内存可见性。
1. 对volatile变量执行写操作时，会在写操作后加入一条store屏障指令。
2. 对volatile变量执行读操作时，会在读操作前加入一条load屏障指令。

## 线程读/写volatile变量的过程
线程写volatile变量的过程：
1. 改变线程工作内存的中volatile变量副本的值。
2. 将改变后的副本的值从工作内存刷新到主内存。

线程读volatile变量的过程：
1. 从主内存中读取volatile变量的最新值到线程的工作内存中。
2. 从工作内存中读取volatile变量的副本。

## 保证变量原子性的方法
1. synchronized关键字（注意加锁的粒度尽量小）
2. ReentrantLock可重入锁
3. AtomicInteger原子类

## volatile使用场景
1. volatile下次值不能与当前值有依赖关系，比如不能有`num`或`number = number * 1`
2. 不能与其他的volatile变量有固定表达式的关系，比如`volatile1 > volatile2`

## volatile与synchronized比较
1. volatile不需要加锁，比synchronized更轻量级，不会阻塞线程

2. 从内存可见性角度，volatile读相当于加锁，volatile写相当于解锁

3. synchronized既能保证可见性，又能保证原子性；而volatile只能保证可见性，无法保证原子性

# 使用JStack生成线程快照
我们可以使用JStack来生成线程快照，来查看当前JVM中的线程信息：
```bash
jstack -l pid
```
在输出信息中，我们可以看到主线程（main）、守护线程的执行状态、各线程锁的持有以及GC线程等信息。
```bash
"Thread-0" #10 daemon prio=5 os_prio=0 tid=0x00007f6ec413a000 nid=0x25d4 waiting on condition [0x00007f6ea1b6b000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at com.sunnywr.DaemonThread.writeToFile(DaemonThreadDemo.java:28)
	at com.sunnywr.DaemonThread.run(DaemonThreadDemo.java:13)
	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
	- None

"main" #1 prio=5 os_prio=0 tid=0x00007f6ec400a800 nid=0x25c4 runnable [0x00007f6ecbb31000]
   java.lang.Thread.State: RUNNABLE
	at java.io.FileInputStream.readBytes(Native Method)
	at java.io.FileInputStream.read(FileInputStream.java:255)
	at java.io.BufferedInputStream.read1(BufferedInputStream.java:284)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:345)
	- locked <0x00000000d711c8c8> (a java.io.BufferedInputStream)
	at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
	at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
	at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
	- locked <0x00000000d72dd658> (a java.io.InputStreamReader)
	at java.io.InputStreamReader.read(InputStreamReader.java:184)
	at java.io.Reader.read(Reader.java:100)
	at java.util.Scanner.readInput(Scanner.java:804)
	at java.util.Scanner.next(Scanner.java:1369)
	at com.sunnywr.DaemonThreadDemo.main(DaemonThreadDemo.java:42)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)

   Locked ownable synchronizers:
	- None
```
