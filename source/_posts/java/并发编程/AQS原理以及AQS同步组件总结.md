---
title: AQS原理以及AQS同步组件总结
date: 2022-03-12
categories:
- Java
tags:
- Java并发编程
language: zh-CN
toc: true
---

### AQS介绍

AQS全称（AbstractQueuedSynchronizer）,即抽象队列同步器。该类位于`java.util.concurrent.locks`包下。

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20220305214816395.png)

AQS是一个抽象类，主要用来构建锁和同步器。AQS 为构建锁和同步器提供了一些通用功能的是实现，因此，使用 AQS 能简单且高效地构造出应用广泛的大量的同步器，比如 `ReentrantLock`，`Semaphore`，其他的诸如 `ReentrantReadWriteLock`，`SynchronousQueue`，`FutureTask`(jdk1.7) 等等都是基于 AQS 的。

<!--more-->

### AQS原理

AQS的核心思想是，如果请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用**CLH**队列锁来实现的，即将暂时获取不到锁的线程加入到队列中。

> CLH同步队列是一个FIFO的双向队列，AQS通过它来完成同步状态的管理，当前线程如果获取同步状态失败时，AQS会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会将首节点唤醒（公平锁），使其再次尝试获取同步状态。

AQS原理图如下所示：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20220305220429846.png)

AQS使用一个int成员变量来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

```java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性
```

状态信息通过 `protected` 类型的`getState()`，`setState()`，`compareAndSetState()` 进行操作：

```java
//返回同步状态的当前值
protected final int getState() {
        return state;
}
 // 设置同步状态的值
protected final void setState(int newState) {
        state = newState;
}
//原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

#### AQS 对资源的共享方式

1. **Exclusive（独占）**

   只有一个线程能执行，如`ReentrantLock`（可重入锁）。又可以分为公平锁和非公平锁，`ReentrantLock`同时支持两种锁，下面以`ReentrantLock`为例介绍公平锁和非公平锁。

   - 公平锁：按照线程在队列中的排队顺序，先到先得
   - 非公平锁：当线程要获取锁时，先通过两次CAS操作去强锁，如果没抢到，当前线程再加入到队列中等待唤醒。

   `ReentrantLock`源码分析：

   ReentrantLock考虑更好的性能，默认采用非公平锁，通过构造方法传入`boolean`来决定是否用公平锁。

   ```java
   /** Synchronizer providing all implementation mechanics */
   private final Sync sync;
   public ReentrantLock() {
       // 默认非公平锁
       sync = new NonfairSync();
   }
   public ReentrantLock(boolean fair) {
       sync = fair ? new FairSync() : new NonfairSync();
   }
   ```

   `ReentrantLock`中公平锁的`lock`方法

   ```java
   static final class FairSync extends Sync {
       final void lock() {
           acquire(1);
       }
       // AbstractQueuedSynchronizer.acquire(int arg)
       public final void acquire(int arg) {
           if (!tryAcquire(arg) &&
               acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
               selfInterrupt();
       }
       protected final boolean tryAcquire(int acquires) {
           final Thread current = Thread.currentThread();
           int c = getState();
           if (c == 0) {
               // 1. 和非公平锁相比，这里多了一个判断：是否有线程在等待
               if (!hasQueuedPredecessors() &&
                   compareAndSetState(0, acquires)) {
                   setExclusiveOwnerThread(current);
                   return true;
               }
           }
           else if (current == getExclusiveOwnerThread()) {
               int nextc = c + acquires;
               if (nextc < 0)
                   throw new Error("Maximum lock count exceeded");
               setState(nextc);
               return true;
           }
           return false;
       }
   }
   ```

   非公平锁的`lock`方法

   ```java
   static final class NonfairSync extends Sync {
       final void lock() {
           // 2. 和公平锁相比，这里会直接先进行一次CAS，成功就返回了
           if (compareAndSetState(0, 1))
               setExclusiveOwnerThread(Thread.currentThread());
           else
               acquire(1);
       }
       // AbstractQueuedSynchronizer.acquire(int arg)
       public final void acquire(int arg) {
           if (!tryAcquire(arg) &&
               acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
               selfInterrupt();
       }
       protected final boolean tryAcquire(int acquires) {
           return nonfairTryAcquire(acquires);
       }
   }
   /**
    * Performs non-fair tryLock.  tryAcquire is implemented in
    * subclasses, but both need nonfair try for trylock method.
    */
   final boolean nonfairTryAcquire(int acquires) {
       final Thread current = Thread.currentThread();
       int c = getState();
       if (c == 0) {
           // 这里没有对阻塞队列进行判断
           if (compareAndSetState(0, acquires)) {
               setExclusiveOwnerThread(current);
               return true;
           }
       }
       else if (current == getExclusiveOwnerThread()) {
           int nextc = c + acquires;
           if (nextc < 0) // overflow
               throw new Error("Maximum lock count exceeded");
           setState(nextc);
           return true;
       }
       return false;
   }
   
   ```

   总结：公平锁和非公平锁只有两处不同：

   1. 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果此时锁没有被占用，就可以直接获取到锁返回。
   2. 非公平锁在 CAS 失败后，和公平锁一样都会进入到 `tryAcquire` 方法，在 `tryAcquire` 方法中，如果发现锁这个时候被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，而是排队。

   公平锁和非公平锁就这两点区别，如果这两次 CAS 都不成功，那么后面非公平锁和公平锁是一样的，都要进入到阻塞队列等待唤醒。

   相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致在阻塞队列中的线程长期处于饥饿状态。
   
2. **Share（共享）**

   多个线程可以同时执行，如`Semaphore/CountDownLatch`。`Semaphore`、`CourtDownLatch`、`CyclicBarrier`、`ReadWriteLock`下面介绍。

   `ReentrantReadWriteLock`读写锁允许多个线程同时对某一资源进行读操作。不同的自定义同步器争用共享资源的方式不同。自定义同步器在实现的时候只需要实现共享资源state（状态）的获取与释放即可，而具体线程等待队列的维护（比如获取资源失败入队、唤醒出队等），AQS已经在上层实现。

#### AQS底层使用模板方法模式

同步器的设计是基于模板方法的。

1. 使用者继承`AbstractQueuedSynchronizer`并且重写指定的方法。（即对共享资源state的获取与释放）。
2. 将AQS组合在自定义同步组件的视线中，并且调用其模板方法，而这些模板方法会调用使用者重写的方法。

这与实现接口的方式不同，AQS是模板方法模式的一个经典应用。

AQS使用了模板方法模式，自定义同步器需要重写下面几个AQS提供的钩子方法：

```java
protected boolean tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
protected boolean tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
protected boolean tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
protected boolean tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
protected boolean isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
```

**什么是钩子方法？**钩子方法是一种被声明在抽象类中的方法，一般使用`protected`关键词修饰，它可以使空方法（由子类是实现），也可以是默认实现的方法。模板设计模式通过钩子方法控制固定步骤的实现。对于模板方法模式的详细介绍参考https://mp.weixin.qq.com/s/zpScSCktFpnSWHWIQem2jg

除了上面要重写的钩子方法，AQS类中的其他方法都是final，所以无法被其他子类重写。

以`ReentrantLock`为例，state初始化为0，表示未锁定状态。A线程`lock()`时，会调用`tryAcquire()`独占该锁并且将`state+1`。此后其他线程再`tryAcquire()`就会失败，直到A线程`unlock()`将`state=0`（即释放锁），其它线程才有机会获得该锁。当前，释放锁之前，A线程可以重复获取该锁（`state的值会累加`）,这就是可重入锁。需要注意获取多少次的锁就需要释放多少次的锁，否则`state无法回到零态`，其它线程将永远无法获得该锁。

以`CountDownLatch`为例，任务分为N个子线程去执行，state也初始化为N（N与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后执行`countDown()`方法，state会CAS地减1。等到所有子线程都执行完后（此时state=0），会执行`unpack()`方法调用主线程，然后 主线程会从`await()`函数返回，继续后续动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式。它们也只需要实现钩子方法中独占方式或者是共享方式中的一种。当然AQS也支持自定义同步器同时实现独占和共享两种方式，比如`ReentrantReadWriteLock`（读写锁）。

下面是两篇有关于AQS 原理和相关源码分析的文章：

- [Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)
- [Java并发包基石-AQS详解](https://www.cnblogs.com/chengxiao/p/7141160.html)

#### Semaphore(信号量)

`synchronized`和`ReentrantLock`都是一次只允许一个线程访问某个资源，`Semaphore`（信号量）可以指定多个线程同时访问某个资源。

示例代码如下：

```java
public class SemaphoreExample1 {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    // 一次只能允许执行的线程数量。
    final Semaphore semaphore = new Semaphore(20);

    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          semaphore.acquire();// 获取一个许可，所以可运行线程数量为20/1=20
          test(threadnum);
          semaphore.release();// 释放一个许可
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }

      });
    }
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}
```

执行`acquire()`方法阻塞，直到有一个许可证可以获得然后拿走一个许可证；每个`release`方法增加一个许可证，这可能会释放一个阻塞的`acquire()`方法。然而，其实并没有实际的许可证对象，`Semqphore`只是维持了一个可获得许可证的数量。`Semaphore`经常用于选址获取某种资源的线程数量。

当然一次也可以获取或者释放多个许可，如下：

```java
semaphore.acquire(5);// 获取5个许可，所以可运行线程数量为20/5=4
test(threadnum);
semaphore.release(5);// 释放5个许可
```

除了 `acquire()` 方法之外，另一个比较常用的与之对应的方法是 `tryAcquire()` 方法，该方法如果获取不到许可就立即返回 false。

`Semaphore` 有两种模式，公平模式和非公平模式。

- **公平模式：** 调用 `acquire()` 方法的顺序就是获取许可证的顺序，遵循 FIFO；
- **非公平模式：** 抢占式。

`Semaphore` 对应的两个构造方法如下：

```java
 // 需要提供许可证数量   
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
// 需要提供许可证数量以及是否为公平锁，默认非公平锁 
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

`Semaphore` 与 `CountDownLatch` 一样，也是共享锁的一种实现。它默认构造 AQS 的 state 为 `permits`。当执行任务的线程数量超出 `permits`，那么多余的线程将会被放入阻塞队列 Park,并自旋判断 state 是否大于0。只有当 state 大于0的时候，阻塞的线程才能继续执行,此时先前执行任务的线程继续执行 `release()` 方法，`release()` 方法使得 state 的变量会加1，那么自旋的线程便会判断成功。 如此，每次只有最多不超过 `permits` 数量的线程能自旋成功，便限制了执行任务线程的数量。

### CountDownLatch（倒计时器）

`CountDownLatch`允许count个线程阻塞，直到所有线程都执行完毕。

`CountDownLatch`是共享锁的一种实现，它默认构造AQS的state值为count。当线程调用`countDown()`方法的时候，其实是使用了`tryReleaseShared()`方法以CAS的操作来减少state，直到state为0。当调用`await()`方法的时候，如果state不为0，那就证明任务还没有执行完毕，`await()`方法会一直阻塞，因此`await()`方法之后的语句不会执行。然后，`CountDownLatch`会自选CAS判断`state==0`，如果等于0，就会释放所有等待的线程，`await()`方法之后的语句得到执行。

#### CountDownLatch的典型用法

1. **某以线程再开始允许前等待n个线程执行完毕**

   将`CountDownLatch`的计数器初始化为n，每当一个线程执行完毕，就会将计数器减1（调用`countdownlatch.countDown()`），当计数器的值为0时，再`CountDownLatch`上`await()`的线程就会被唤醒。典型的应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

2. **实现多个线程开始执行任务的最大并行性**

   并行强调多个线程在某一时刻同时执行。做法是初始化一个共享的`CountDownLatch`对象，将其计数器初始化为1（即`newCOuntDownLatch(1)`），多个线程在开始执行任务前首先执行`countdownlatch.await()`，当主线程调用`countDown()`时，计数器变为0，多个线程同时被唤醒。

示例代码如下所示：

```java
public class CountDownLatchExample1 {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          test(threadnum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } finally {
          countDownLatch.countDown();// 表示一个请求已经被完成
        }

      });
    }
    countDownLatch.await();
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}
```

上述代码定义了请求的数量为550，当这550个请求被处理完成之后，才会执行`System.out.println("finsh")`。

与`CountDownLatch`的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用`CountDownLatch.await()`方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

在使用`await()`方法的时候一定要注意死锁问题，如果不能释放count个线程，那么`await()`方法将会一直阻塞。

#### CountDownLatch的不足

`CountDownLatch`是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当`CountDownLatch`使用完毕之后，它不能被再次使用。

#### CountDownLatch常见面试题

- `CountDownLatch` 怎么用？应用场景是什么？
- `CountDownLatch` 和 `CyclicBarrier` 的不同之处？
- `CountDownLatch` 类中主要的方法？

### CyclicBarrier(循环栅栏)

`CyclicBarrier`和`CountDownLatch`非常类似，它也可以实现线程间的技术等待，但是它的功能比`CountDownLatch`更加强大。主要应用常见和`CountDownLatch`类似。

> `CountDownLatch`的实现是基于AQS的，而`CyclicBarrier`是基于`ReentrantLock`（`ReentrantLock`也属于AQS同步器)和`Condition`的。

`CyclicBarrier`是可循环使用的屏障(`Barrier`)。它要作的事情是：让一组线程到达一个屏障（也可以成为同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会打开， 所有被屏障拦截的线程才会继续执行。

`CyclicBarrier`的默认构造方法是`CyclicBarrier(int parties)`，如下所示：

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

其参数表示屏障拦截的线程数量，每个线程调用`await()`方法告诉`CyclicBarrier`已经抵达屏障，然后阻塞等待。

#### CyclicBarrier 的应用场景

`CyclicBarrier` 可以用于多线程计算数据，最后合并计算结果的应用场景。比如我们用一个 Excel 保存了用户所有银行流水，每个Sheet保存一个帐户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet 的日均银行流水，最后，再用 barrierAction 用这些线程的计算结果，计算出整个 Excel 的日均银行流水。

#### CyclicBarrier 使用示例

```java
public class CyclicBarrierExample2 {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum:" + threadnum + "is ready");
    try {
      /**等待60秒，保证子线程完全执行结束*/
      cyclicBarrier.await(60, TimeUnit.SECONDS);
    } catch (Exception e) {
      System.out.println("-----CyclicBarrierException------");
    }
    System.out.println("threadnum:" + threadnum + "is finish");
  }
}
```

运行结果，如下：

```java
threadnum:0is ready
threadnum:1is ready
threadnum:2is ready
threadnum:3is ready
threadnum:4is ready
threadnum:4is finish
threadnum:0is finish
threadnum:1is finish
threadnum:2is finish
threadnum:3is finish
threadnum:5is ready
threadnum:6is ready
threadnum:7is ready
threadnum:8is ready
threadnum:9is ready
threadnum:9is finish
threadnum:5is finish
threadnum:8is finish
threadnum:7is finish
threadnum:6is finish
```

可以看到当线程数量也就是请求数量达到我们定义的5个的时候，`await()`方法之后的方法才会被执行。

另外，`CyclicBarrier`还提供一个更高级的构造函数`CyclicBarrier(int parties,Runnable barrierAction)`，用于在线程到达屏障时，优先执行`barrierAction`，方便处理更复杂的业务场景。示例代码如下：

```java
public class CyclicBarrierExample3 {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
    System.out.println("------当线程数达到之后，优先执行------");
  });

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum:" + threadnum + "is ready");
    cyclicBarrier.await();
    System.out.println("threadnum:" + threadnum + "is finish");
  }
}
```

运行结果如下：

```java
threadnum:0is ready
threadnum:1is ready
threadnum:2is ready
threadnum:3is ready
threadnum:4is ready
------当线程数达到之后，优先执行------
threadnum:4is finish
threadnum:0is finish
threadnum:2is finish
threadnum:1is finish
threadnum:3is finish
threadnum:5is ready
threadnum:6is ready
threadnum:7is ready
threadnum:8is ready
threadnum:9is ready
------当线程数达到之后，优先执行------
threadnum:9is finish
threadnum:5is finish
threadnum:6is finish
threadnum:8is finish
threadnum:7is finish
......
```

#### CyclicBarrier源码分析

当调用`CyclicBarrier`对象调用`await()`方法时，实际上调用的是`dowait(fasle,0L)`方法。`await()`方法会阻塞线程，只有当阻塞的线程数量达到`parties`的值时，栅栏才会打开，线程才能继续执行。

```java
public int await() throws InterruptedException, BrokenBarrierException {
  try {
    	return dowait(false, 0L);
  } catch (TimeoutException toe) {
   	 throw new Error(toe); // cannot happen
  }
}
```

`dowait(fasle,0L)`：

```java
    // 当线程数量或者请求数量达到 count 时 await 之后的方法才会被执行。上面的示例中 count 的值就为 5。
    private int count;
    /**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        // 锁住
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            // 如果线程中断了，抛出异常
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            // cout减1
            int index = --count;
            // 当 count 数量减为 0 之后说明最后一个线程已经到达栅栏了，也就是达到了可以执行await 方法之后的条件
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    // 将 count 重置为 parties 属性的初始化值
                    // 唤醒之前等待的线程
                    // 下一波执行开始
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

总结：`CyclicBarrier` 内部通过一个 count 变量作为计数器，count 的初始值为 parties 属性的初始化值，每当一个线程到了栅栏这里了，那么就将计数器减一。如果 count 值为 0 了，表示这是这一代最后一个线程到达栅栏，就尝试执行我们构造方法中输入的任务。

#### CyclicBarrier和CountDownLatch的区别

`CountDownLatch`是计数器，只能使用一次，而`CyclicBarrier`的计数器提供`reset`功能，可以多次使用。在javadoc中是这么描述的：

>CountDownLatch: A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.(CountDownLatch: 一个或者多个线程，等待其他多个线程完成某件事情之后才能执行；)
>CyclicBarrier : A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point.(CyclicBarrier : 多个线程互相等待，直到到达同一个同步点，再继续一起执行。)

对于`CountDownLatch`来说，重点是一个线程（多个线程）等待，而其他的N个线程在完成”某件事情“之后，可以终止，也可以等待。而对于`CyclicBarrier`，重点是多个线程，在任意一个线程没有到达”栅栏“位置，所有的线程都必须等待。

`CountDownLatch`是计数器，并且是递减地计数，而`CyclicBarrier`是一个阀门，需要所有的线程都到达，阀门才能打开，然后继续执行。

#### ReentrantLock 和 ReentrantReadWriteLock

这个在上文已经介绍过，需要注意的是，读写锁 `ReentrantReadWriteLock` 可以保证多个线程可以同时读，所以在读操作远大于写操作的时候，读写锁就非常适合。



### CLH同步队列原理

CLH同步队列结构图如下所示：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20220305234249027.png)

```java
    static final class Node {
        /** 节点正在共享模式下 */
        static final Node SHARED = new Node();
        /** 节点正在独占模式下 */
        static final Node EXCLUSIVE = null;

        /** waitStatus值，表示线程已取消。超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；*/
        static final int CANCELLED =  1;
        /** waitStatus值。后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行 */
        static final int SIGNAL    = -1;
        /** waitStatus值，表示线程赈灾等待条件。节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中 */
        static final int CONDITION = -2;
        /**
         * 表示下一次共享式同步状态获取将会无条件地传播下去
         */
        static final int PROPAGATE = -3;

      
    	/** 等待状态 */
        volatile int waitStatus;

        /** 前驱节点 */
        volatile Node prev;

        /** 后继节点 */
        volatile Node next;

        /** 获取同步状态的线程 */
        volatile Thread thread;

        /** 链接到下一个等待条件的节点，或共享的特殊值。因为条件队列只有在独占模式下保持时才被访问，所以我们只需要一个简单的链接队列来在节点等待条件时保持节点。然后，它们被转移到队列中重新获取。由于条件只能是独占的，我们通过使用特殊值来指示共享模式来保存字段。*/
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        /** Establishes initial head or SHARED marker. */
        Node() {}

        /** Constructor used by addWaiter. */
        Node(Node nextWaiter) {
            this.nextWaiter = nextWaiter;
            THREAD.set(this, Thread.currentThread());
        }

        /** Constructor used by addConditionWaiter. */
        Node(int waitStatus) {
            WAITSTATUS.set(this, waitStatus);
            THREAD.set(this, Thread.currentThread());
        }

        /** CASes waitStatus field. */
        final boolean compareAndSetWaitStatus(int expect, int update) {
            return WAITSTATUS.compareAndSet(this, expect, update);
        }

        /** CASes next field. */
        final boolean compareAndSetNext(Node expect, Node update) {
            return NEXT.compareAndSet(this, expect, update);
        }

        final void setPrevRelaxed(Node p) {
            PREV.set(this, p);
        }

        // VarHandle mechanics
        private static final VarHandle NEXT;
        private static final VarHandle PREV;
        private static final VarHandle THREAD;
        private static final VarHandle WAITSTATUS;
        static {
            try {
                MethodHandles.Lookup l = MethodHandles.lookup();
                NEXT = l.findVarHandle(Node.class, "next", Node.class);
                PREV = l.findVarHandle(Node.class, "prev", Node.class);
                THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
                WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
            } catch (ReflectiveOperationException e) {
                throw new ExceptionInInitializerError(e);
            }
        }
    }
```

#### 入列

通过调用`addWaiter()`方法执行入队操作，源码如下：

```java
    private Node addWaiter(Node mode) {
        //新建Node
        Node node = new Node(Thread.currentThread(), mode);
        //快速尝试添加尾节点
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            //CAS设置尾节点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //多次尝试
        enq(node);
        return node;
    }
```

`addWaiter(Node node)`先通过快速尝试设置尾节点，如果失败，则调用`enq(Node node)`方法设置尾节点：

```java
    private Node enq(final Node node) {
        //多次尝试，直到成功为止
        for (;;) {
            Node t = tail;
            //tail不存在，设置为首节点
            if (t == null) {
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //设置为尾节点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

在上面代码中，两个方法都是通过一个CAS方法`compareAndSetTail(Node expect, Node update)`来设置尾节点，该方法可以确保节点是线程安全添加的。在`enq(Node node)`方法中，AQS通过“死循环”的方式来保证节点可以正确添加，只有成功添加后，当前线程才会从该方法返回，否则会一直执行下去。

#### 出列

CLH同步队列线程FIFO，首节点的线程释放同步状态后，将会唤醒它的后继节点(next)，而后继节点将会在获取同步状态成功时将会设置为首节点。head执行该节点并断开原来的首节点的next和当前节点的prev，注意在这个过程中时不需要使用CAS来保证的，因为只有一个线程能够成功获取同步状态。过程如下图所示：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20220307170846779.png)
