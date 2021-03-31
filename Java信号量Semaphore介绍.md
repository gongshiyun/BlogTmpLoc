# Java信号量Semaphore介绍

## 信号量（Semaphore）简介

信号量（Semaphore）是对锁的扩展。相对于synchronized和ReentrantLock，每次只允许一个线程访问一个资源，而信号量不一样，信号量可以控制多个线程访问同一个资源，通过提供有限的许可，限制同时有多少个线程可以访问某个资源。

Semaphore类结构：

![image](https://user-images.githubusercontent.com/15052000/113153132-5f1eaf80-9269-11eb-87c9-ec44dbd7cf51.png)

## 使用场景

例如数据库的连接池，限制10个连接，可以使用Semaphore，每个需要连接数据库的线程都需要向Semaphore获取一个许可，获取成功后才能连接数据库；获取失败则阻塞等待。线程连接数据库执行完逻辑后，将许可放回Semaphore，以便其他线程获取连接。



## 使用示例

Semaphore最基本的使用，就是通过acquire()方法获取一个许可，如果获取不了，线程会阻塞直到获取成功，然后对资源进行访问，访问完毕后，调用release方法释放许可，以下demo：

```java
public class SemaphoreDemo implements Runnable {

    // 初始化信号量，提供5个许可
    private final Semaphore semaphore = new Semaphore(5);

    @Override
    public void run() {
        try {
            // 获取许可，获取不到则阻塞
            semaphore.acquire();
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getId() + " done at " + LocalTime.now());
            // 释放许可
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        SemaphoreDemo semaphoreDemo = new SemaphoreDemo();
        // 20个线程同时竞争资源
        for (int i = 0; i < 20; i++) {
            executorService.execute(semaphoreDemo);
        }
    }
}

运行main方法，输出如下：
16 done at 20:29:21.661
15 done at 20:29:21.661
12 done at 20:29:21.661
14 done at 20:29:21.661
11 done at 20:29:21.661
17 done at 20:29:23.668
21 done at 20:29:23.668
18 done at 20:29:23.668
19 done at 20:29:23.668
13 done at 20:29:23.668
22 done at 20:29:25.679
29 done at 20:29:25.679
28 done at 20:29:25.679
27 done at 20:29:25.679
23 done at 20:29:25.679
30 done at 20:29:27.688
24 done at 20:29:27.688
26 done at 20:29:27.688
25 done at 20:29:27.688
20 done at 20:29:27.688
```

可见每两秒有5个线程执行输出，验证了Semaphore的功能。



## 信号量原理分析

Semaphore的内部实现依赖了AQS来实现。

Sync类继承了AQS，FairSync和NonfairSync为Sync的子类，分别实现公平和非公平的逻辑。

信号量提供了两个构造方法：

```java
// permits为许可的数量,该构造函数对应为非公平模式的信号量
public Semaphore(int permits);
// 第二个参数可以设置公平模式的信号量
public Semaphore(int permits, boolean fair);
```

构造方法内部实现是调用FairSync和NonfairSync的构造方法，底层是通过Sync的构造方法初始化AQS的共享变量state，通过state来表明可用的许可数量：

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
NonfairSync(int permits) {
    super(permits);
}
Sync(int permits) {
	setState(permits);
}
```

后续通过acquire和release方法来获取和归还许可。

### 公平模式和非公平模式

公平模式和非公平模式实现了AQS的tryAcquireShared方法来获取许可，

公平模式下的获取许可的逻辑：

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        // 判断是否有线程在等待许可，有则返回-1
        if (hasQueuedPredecessors())
            return -1;
        // 可用的许可数
        int available = getState();
        // 获取许可后剩余的许可数
        int remaining = available - acquires;
        if (remaining < 0 ||
            // 通过CAS更新state值
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

首先会去判断是否有线程在等待获取许可，如果有就返回-1。

非公平模式下获取许可逻辑：

```java
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        // 可用的许可数
        int available = getState();
        // 获取许可后剩余的许可数
        int remaining = available - acquires;
        if (remaining < 0 ||
            // 通过CAS更新state值
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

在非公平模式下，不会管是否有线程等待许可，直接通过CAS去抢许可，谁抢到就是谁的。

### 获取许可

获取许可的方法acquire()：

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        // 如果获取许可失败了，放入AQS等待队列中获取许可
        doAcquireSharedInterruptibly(arg);
}
```

获取许可的方法实际调用的是Sync的基类AQS的acquireSharedInterruptibly()方法，内部又调用了子类实现的tryAcquireShared()方法，即上一节提到的公平锁和非公平锁的tryAcquireShared方法。



###  归还许可

归还许可对应release方法：

```java
public void release() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
	return false;
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
        	throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
        	return true;
    }
}
```

获取许可通过减少state的值实现，而归还许可就是通过增加state的值来实现。
