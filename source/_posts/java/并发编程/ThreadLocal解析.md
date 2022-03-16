---
title: ThreadLocal详解
date: 2022-03-16
categories:
- Java
tags:
- Java并发编程
language: zh-CN
toc: true
---

### ThreadLocal解析

> 本文参照[一枝花算不算浪漫](https://juejin.im/post/5eacc1c75188256d976df748)

ThreadLocal的特点：

- 线程并发：在多线程场景下
- 传递数据：可以通过ThreadLocal在同一线程，不同组件中传递公共变量
- 线程隔离：每个线程的变量都是独立的，不会相互影响

主要探讨以下问题：

- `ThreadLocal`的key是弱引用，那么在`ThreadLocal.get()`的时候，发生GC之后，key是否为null？
- `ThreadLocal中ThreadLocalMap`的数据结构？
- `ThreadLocalMap`的Hash算法？
- `ThreadLocalMap`的扩容机制？
- `ThreadLocalMap`中过期key的清理机制？探测式清理和启发式清理流程？
- `ThreadLocalMap.set()`方法实现原理？
- `ThreadLocalMap.get()`方法实现原理？

<!--more-->

#### ThreadLocal代码演示

首先运行下面的代码：

```java
public class ThreadLocalTest{
    ThreadLocal<String> t1 = new ThreadLocal<>();
    private String content;
    private String getContent(){
        return t1.get();
    }

    private void setContent(String content){
        t1.set(content);
    }

    public static void main(String[] args) {
        ThreadLocalTest test = new ThreadLocalTest();
        for(int i=0;i<5;i++){
            Thread thread =new Thread(new Runnable() {
                @Override
                public void run() {
                    test.setContent(Thread.currentThread().getName()+"的数据");
                    System.out.println(Thread.currentThread().getName()+"--->"+test.getContent());
                }
            });
            thread.setName("线程"+i);
            thread.start();
        }
    }
}
```

输出结果如下：

```
线程0--->线程0的数据
线程2--->线程2的数据
线程1--->线程1的数据
线程4--->线程4的数据
线程3--->线程3的数据
```

如上，`ThreadLocal`实现了线程隔离，在多线程并发的场景下，每个线程中的变量都是相互独立的。`ThreadLocal`对象可以提供线程局部变量，每个线程`Thread`拥有一份自己的**副本变量**，多个线程互不干扰。

#### ThreadLocal与synchronized的区别

 synchronized也同样可以实现上述代码的结果，但是程序的性能会大大降低，原本并行的程序可能会变成串行。

|        |                         synchroniezd                         |                         ThreadLocal                          |
| :----: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  原理  | 同步机制采用“以空间换时间”的方式，只提供了一份变量，让不同的线程排队访问。 | ThreadLocal采用“以空间换时间”的方式，为每一个线程都提供了一份变量的副本，从而实现同时访问而互不干扰 |
| 关注点 |               多个线程之间访问同一个资源的同步               |                 ··多个线程之间的数据相互隔离                 |

#### `ThreadLocal`的数据结构

![image-20220315171354835](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220315171354835.png)

`ThreadLocalMap`有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，也就是说每个线程有一个自己的`ThreadLocalMap`。可以这么理解，ThreadLocalMap的key是ThreadLocal，value是代码中放入的值（实际上key并不是ThreadLocal本身，而是ThreadLocal的一个**弱引用**）。每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的map找对应的key，以此实现了线程隔离。`ThreadLocalMap`有点类似`HashMap`结构，只是`HashMap`是由数组+链表实现的，而`ThreadLocalMap`中并没有链表结构。并且它的`Entry`，key是`ThreadLocal<?> k`，继承自`WeakReference`，也就是弱引用类型。

### GC之后key是否为null？

由于ThreadLocal的key是弱引用，那么在`ThreadLocal.get()`的时候，发生在`GC`之后，key的值是否为null？

**Java的四种引用类型**

- 强引用：

  强引用是使用最普遍的引用，如果一个对象具有强引用，那么垃圾回收器绝不会回收它。如下：

  ```java
  Object strongReference =new Object()
  ```

  当内存空间不足时，Java虚拟机宁愿抛出`OutOfMemoryError`错误，使得程序抛出异常而终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。如果强引用对象不使用时，需要弱化从而使得GC能够回收，如下所示：

  ```java
  strongReference=null
  ```

  由开发者显式地设置`strongReference`对象为null，或者让其超出对象的生命周期范围，则GC认为该对象不存在引用，这时就可以回收这个对象。当然具体什么时候回收取决所使用的垃圾回收算法。

  在一个方法的内部都有一个强引用，这个引用保存在Java**栈**中，而真正引用的内容(Object对象)保存在Java**堆**中，当这个方法运行完成后，则会退出**方法栈**，则引用对象的引用数为0，则这个对象会被回收。但是如果这个`strongReference`是**全局变量**时，就需要在不用这个对象时赋值为null，因为强引用不会被垃圾回收。

  例如**ArrayList**的clear方法：

  在ArrayList类中定义了一个elementData数组，在调用clear方法清空数组时，每个数组元素被赋值为null。不同于`elementData=null`，强引用依然存在，为了避免后续调用`add()`等方法添加元素时进行内存的重新分配。使用`clear()`方法只是清空了数组中的内容，但是数组的引用依然存在。因此使用如`clear()`方法清除内存数组中存放的引用类型进行内存释放特别适用，这样就可以及时释放内存。

- 软引用:

  如果一个对象只有软引用，那么内存空间充足时，垃圾回收就不会回收它；如果内存空间不足，那么垃圾回收就会回收它们。因此软引用可以用来实现内存敏感的告诉缓存。

  ```java
  	// 强引用
  	String strongReference = new String("abc");
  	// 软引用
  	String str = new String("abc");
  	SoftReference<String> softReference = new SoftReference<String>(str);
  ```

  软引用可以和一个引用队列(`ReferenceQueue`)联合使用，如果软引用所引用的对象被垃圾回收，java虚拟机会把这个引用加入到与之关联的引用队列中。

  ```java
  	ReferenceQueue<String> referenceQueue = new ReferenceQueue<>();
      String str = new String("abc");
      SoftReference<String> softReference = new SoftReference<>(str, referenceQueue);
  
      str = null;
      // Notify GC
      System.gc();
  
      System.out.println(softReference.get()); // abc
  
      Reference<? extends String> reference = referenceQueue.poll();
      System.out.println(reference); //null
  ```

  > 注意jvm即使扫描到软引用对象也不一定会回收它，软引用对象只有在虚拟机内存不够的时候才会被回收。当内存不足时，JVM首先将软引用中的对象设置为null，然后通知垃圾回收器进行回收。也就是说，垃圾回收器会在虚拟机抛出OOM之前回收软引用对象，而且虚拟机会尽可能优先回收长时间闲置不用的软引用对象。对于那些刚构建的软引用对象或者那些刚刚被使用的软引用对象则尽可能保留，这就是引入引用队列的原因。

  应用场景：

  浏览器的后退按钮。后退时，这个后退时显示的网页内容是重新进行请求获取还是在缓存中获取。

  如果一个网页在浏览结束的时候就进行“垃圾回收”，则按后退查看之前浏览过的网页则需要重新构建；如果将浏览过的网页全部存储在内存中就会造成大量的浪费，甚至会造成内存溢出。这个时候就可以借鉴软引用的思想，将浏览完的网页设置为软引用，在垃圾回收的时候尽量保留那些新打开的页面或者最近被使用的页面。

- 弱引用：

  弱引用和软引用的区别就是只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了弱引用对象的存在，则不管当前的内存是否足够，都会进行回收。不过由于垃圾回收器是一个优先级很低的线程，因此不一定可以很快地发现弱引用对象。垃圾回收器线程一旦发现了弱引用对象，则会将其设置为null，并且通知垃圾回收器进行回收。同样地，**弱引用**可以和一个**引用队列**(`ReferenceQueue`)联合使用，如果**弱引用**所引用的**对象**被**垃圾回收**，`Java`虚拟机就会把这个**弱引用**加入到与之关联的**引用队列**中。

- 虚引用：

  虚引用和其他几种引用都不同，虚引用不会决定对象的生命周期，如果一个对象持有虚引用，那么它就和没有任何引用一样，任何时候都可能被垃圾回收器回收。

  应用场景：

  虚引用主要用来跟踪对象被垃圾回收的活动。虚引用与软引用和弱引用的区别在于：虚引用必须和引用队列联合使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列之中。在声明一个虚引用的时候需要传入一个引用队列。当虚引用执行完finalize函数的时候就会被加入到队列中。因从可以通过判断引用队列中是否加入了虚引用来判断所引用的对象是否将要被垃圾回收。如果发现某个虚引用已经被加入到引用队列中，那么就可以在其被回收之前完成相应的处理逻辑。

  ```java
      String str = new String("abc");
      ReferenceQueue queue = new ReferenceQueue();
      // 创建虚引用，要求必须与一个引用队列关联
      PhantomReference pr = new PhantomReference(str, queue);
  ```

**总结**

| 引用类型 | 被垃圾回收时间 | 用途               | 生存时间          |
| -------- | -------------- | ------------------ | ----------------- |
| 强引用   | 从来不会       | 对象的一般状态     | JVM停止运行时终止 |
| 软引用   | 当内存不足时   | 对象缓存           | 内存不足时终止    |
| 弱引用   | 正常垃圾回收时 | 对象缓存           | 垃圾回收后终止    |
| 虚引用   | 正常垃圾回收时 | 跟踪对象的垃圾回收 | 垃圾回收后终止    |

接下来来看一下GC后ThreadLocal中的数据情况：

```java
public class ThreadLocalDemo {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InterruptedException {
        Thread t = new Thread(()->test("abc",false));
        t.start();
        t.join();
        System.out.println("--gc后--");
        Thread t2 = new Thread(() -> test("def", true));
        t2.start();
        t2.join();
    }

    private static void test(String s,boolean isGC)  {
        try {
            new ThreadLocal<>().set(s);
            if (isGC) {
                System.gc();
            }
            Thread t = Thread.currentThread();
            Class<? extends Thread> clz = t.getClass();
            Field field = clz.getDeclaredField("threadLocals");
            field.setAccessible(true);
            Object ThreadLocalMap = field.get(t);
            Class<?> tlmClass = ThreadLocalMap.getClass();
            Field tableField = tlmClass.getDeclaredField("table");
            tableField.setAccessible(true);
            Object[] arr = (Object[]) tableField.get(ThreadLocalMap);
            for (Object o : arr) {
                if (o != null) {
                    Class<?> entryClass = o.getClass();
                    Field valueField = entryClass.getDeclaredField("value");
                    Field referenceField = entryClass.getSuperclass().getSuperclass().getDeclaredField("referent");
                    valueField.setAccessible(true);
                    referenceField.setAccessible(true);
                    System.out.println(String.format("弱引用key:%s,值:%s", referenceField.get(o), valueField.get(o)));
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出如下：

```java
弱引用key:java.lang.ThreadLocal@3100959c,值:abc
弱引用key:java.lang.ThreadLocal@76292830,值:java.lang.ref.SoftReference@6c848e0f
--gc后--
弱引用key:null,值:def
```

![image-20220315214308828](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220315214308828.png)

 如图所示，这里创建的ThreadLocal并没有指向任何值，也就是没有引用：

```
new ThreadLocal<>().set(s);
```

 所以这里在GC之后，key就会被垃圾回收器回收，但是如果用一个对象保存ThreadLocal的引用，那么可以看到如下图所示的结果：

![image-20220315221730752](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220315221730752.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 

key并不是null，按照上面介绍的弱引用，垃圾回收，那么在这个时候key得是null。但是没有被回收，就说明存在强引用。如果不存在强引用，那么key就会被回收，而value不会被回收，于是就导致了key被回收，value永远存在的情况，这样就会出现**内存泄漏**。如下图所示，`ThreadLocal`的强引用依然存在：

![image-20220315222301897](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220315222301897.png)

每个`Thread`内部都维护着一个`ThreadLocalMap`的数据结构，map的Key值为`ThreadLocal`，那么当某个`ThreadLocal`对象不再使用（没有被引用时），每个已经关联了此`ThreadLocal`的线程，由于`ThreadLocalMap`内部存储实体结构`Entry<ThreadLocal,T>`继承自`java.lang.ref.WeakReference`，这样当`ThreadLocal`不再被引用时，因为弱引用机制的原因，会进行垃圾回收，也就是其线程内部的`ThreadLocalMap`会释放对`ThreadLocal`的引用从而让jvm回收`ThreadLocal`对象。但是不会回收线程变量中的值`T`对象，所以会内存泄露，但是`ThreadLocal`会在调用`get()`和`set()`方法时都会定期回收无效的Entry。

看一下源码在哪里使用了弱引用：

```java
/**
	 * The entries in this hash map extend WeakReference, using 
	 * its main ref field as the key (which is always a ThreadLocal object). 
	 * Note that null keys (i.e. entry.get() == null) mean that the key is no longer referenced, 
	 * so the entry can be expunged from table. 
	 * Such entries are referred to as "stale entries" in the code that follows.
	 */
	static class Entry extends WeakReference<ThreadLocal<?>> {
		/** The value associated with this ThreadLocal. */
		Object value;
 
		Entry(ThreadLocal<?> k, Object v) {
			super(k);
			value = v;
		}
	}
```

`Entry`中的key是弱引用，key弱指向`ThreadLocal<?>`对象，并且key只是只是ThreadLocal强引用的副本，value是实际对应的对象。当显示地将key所指向的对象设置为null的时候，就只有剩下了key这一个弱引用，GC时会回收掉`ThreadLocal<?>`对象。为什么上面的例子中弱引用和GC都没有导致key作为虚引用被回收，因为它本身被当前线程的Map强引用，只有当不存在线程的强引用之后，这个`weakreference`才会被垃圾回收。

### `ThreadLocal.set()`方法源码详解

![image-20220316191029308](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316191029308.png)

ThreadLocal中set方法的原理如上图所示，整个过程主要就是判断`ThreadLocal`是否存在，然后使用`ThreadLocal`中的`set`方法进行数据处理。其源码如下：

```java
public void set(T value) {
    // 得到当前线程
    Thread t = Thread.currentThread();
    // 得到当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 如果map不为null，直接设置值
        map.set(this, value);
    else
        // 如果map为空，那么就创建map并且设置值
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### `ThreadLocalMap`的Hash算法

源代码如下：

```java
Entry[] tab = table;
int len = tab.length;
int i = key.threadLocalHashCode & (len-1);
```

ThreadLocalMap中hash算法很简单，这里的i就是当前key在散列表中对应的数组下标位置。

这里关键的就是`ThreadLocalHashCode`值的计算，`ThreadLocal`中有一个属性为`HASH_INCREMENT = 0x61c88647`

```JAVA
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    static class ThreadLocalMap {
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
}
```

每次创建一个`ThreadLocal`对象，这个`ThreadLocal.nextHashCode`这个值就会增长`0x61c88647`。这个数称为**斐波那契数**。使用斐波那契数作为hash增量，会使得hash分布非常均匀。如下图所示，分布均匀：

![image-20220316194950390](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316194950390.png)

### `ThreadLocalMap`解决Hash冲突

> 下面的示例图中，绿色块`Entry`代表正常数据，灰色快代表`Entry`的`key`为`null`，已被垃圾回收。白色块代表`Entry`为`null`。

我们知道`HashMap`中解决冲突的方法是在数组上构造一个链表结构，冲突的数据会继续接到链表上，如果超过一定的数量则会将链表结构转化为**红黑树**。

而`ThreadLocalMap`并不存在链表结构，所以其处理hash冲突的方式与`HashMap`并不一样。如下图所示：

![image-20220316200357282](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316200357282.png)

此时需要插入一个值为27的数据，通过哈希计算之后应该放在下标为4的位置，但是下标为4的位置已经有数据了，`ThreadLocalMap`的处理方式是继续向后寻找，一直找到`Entry`为`null`的位置才会停止查找并且将数据放入该位置中。当然在线性查找的过程中，如果遇到了`Entry`不为`null`且`key`值或者`Entry`中值为`null`的情况都会有不同的处理。

上图中有一个`Entry`的`key`为`null`的数据，原因就是前面所提到的因为key的类型是弱引用类型，所以会有这种数据的存在，但是在`set`的过程中，会对这些数据进行清理。

### `ThreadLocalMap.set()`详解

在往ThreadLocalMap中set数据的时候大概会遇到下面的几种情况：

- 要set数据的位置对应的`Entry`为空，这个时候只需要直接放入数据即可。

- 要set数据的位置存在数据，这个时候又要分几种情况：

  - 该位置的key值与当前ThreadLocal通过hash计算获取的key值一致，这个时候直接更新该位置的数据。

  - 在往后遍历的过程中，在找到Entry为null的位置之前，没有遇到key过期的Entry，那么就直接将数据放入Entry为null的位置。

  - 在往后遍历的过程中，在找到key值相等的数据之前，没有遇到key过期的Entry，那么直接更新当前位置的数据。

  - 在往后遍历的过程中，在找到`Entry`为`null`的位置之前，遇到`key`过期的`Entry`，如下图所示，在往后遍历的过程中，在`index=7`位置的`Entry`的`key`为`null`：

    ![image-20220316202329998](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316202329998.png)`index=7`位置`Entry`的`key`已经被垃圾回收，那么就需要进行处理，避免内存泄漏。此时会执行`replaceStaleEntry()`方法，该方法的含义是替换过期数据，以index=7为起点开始扫描，进行探测式数据清理工作。

    初始化探测式清理过期数据扫描的开始位置为：`slotToExpunge = staleSlot = 7`，以当前`staleSlot`开始向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标`slotToExpunge`。for循环迭代，直到碰到`Entry`为`null`结束。如果找到了过期的数据，继续向前迭代，直到遇到`Entry=null`的槽位才停止迭代，如下图所示，**slotToExpunge 被更新为 0**：![image-20220316204044572](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316204044572.png)

    上面向前迭代的操作是为了更新探测清理过期数据的起始下标`slotToExpunge`的值，这个值是用来判断当前过期槽位`staleSlot`之前是否还有过期元素。

    接下来以`staleSlot`位置（`index=7`）向后迭代查找，如果找到了相同`key`值的`Entry`数据，如下图所示：

    ![image-20220316204344307](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316204344307.png)

    则会更新`Entry`的值并交换`slateSlot`元素的位置(`slaleSlot`位置为过期元素)，更新`Entry`数据，然后开始进行过期`Entry`的清理工作（从`slotToExpunge=0`的位置向后检查过期数据并且清理），如下图所示：

    ![image-20220316204658068](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316204658068.png)

    向后遍历的过程中，如果没有找到相同key值的Entry数据：

    ![image-20220316205420141](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316205420141.png)

    从当前节点`staleSlot`向后查找`key`值相等的`Entry`元素，直到`Entry`为`null`则停止寻找。通过上图可知，此时`table`中没有`key`值相同的`Entry`。

    创建新的`Entry`，替换`table[stableSlot]`位置：

    ![image-20220316205705908](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316205705908.png)

    替换完成后也是进行过期元素清理工作，清理工作主要是有两个方法：`expungeStaleEntry()`和`cleanSomeSlots()`，这两个方法在后面会讲到。

### `ThreadLocalMap.set()`源码详解

```java
private void set(ThreadLocal<?> key, Object value) {
    // 通过key来计算在散列表中对应的位置，然后以当前key对应的桶的位置向后查找，找到可以使用的桶。
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

什么样的桶可以被使用？

1. 要set的key和Entry中的key相等，需要替换，可以使用
2. 碰到一个过期的桶，执行替换逻辑，占用过期桶
3. 查找过程中，碰到桶中`Entry=null`的情况，直接使用

for循环中向前向后查找的逻辑是靠`nextIndex()`和`prevIndex()`方法实现，其源码如下：

```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

接着看剩下`for`循环中的逻辑：

1. 遍历当前`key`值对应的桶中`Entry`数据为空，这说明散列数组这里没有数据冲突，跳出`for`循环，直接`set`数据到对应的桶中；
2. 如果`key`值对应的桶中`Entry`数据不为空：
   2.1 如果`k = key`，说明当前`set`操作是一个替换操作，做替换逻辑后返回；
   2.2 如果`key = null`，说明当前桶位置的`Entry`是过期数据，执行`replaceStaleEntry()`方法(核心方法)，然后返回；
3. `for`循环执行完毕，继续往下执行说明上面两种情况都不成立，向后迭代的过程中直到找到`entry`为`null`的情况：
   3.1 在`Entry`为`null`的桶中创建一个新的`Entry`对象
   3.2 执行`++size`操作
4. 调用`cleanSomeSlots()`做一次启发式清理工作，清理散列数组中`Entry`的`key`过期的数据
   4.1 如果清理工作完成后，未清理到任何数据，且`size`超过了阈值(数组长度的 2/3)，进行`rehash()`操作
   4.2 `rehash()`中会先进行一轮探测式清理，清理过期`key`，清理完成后如果**size >= threshold - threshold / 4**，就会执行真正的扩容逻辑(扩容逻辑往后看)。

接着重点看下`replaceStaleEntry()`方法，`replaceStaleEntry()`方法提供替换过期数据的功能，其源代码如下：

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))

        if (e.get() == null)
            slotToExpunge = i;

    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {

        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

`slotToExpunge`表示开始探测式清理过期数据的开始下标，默认从当前的`staleSlot`开始。以当前的`staleSlot`开始，向前迭代查找，找到没有过期的数据，`for`循环一直碰到`Entry`为`null`才会结束。如果向前找到了过期数据，更新探测清理过期数据的开始下标为 i，即`slotToExpunge=i`，其代码逻辑如下代码块：

```java
for (int i = prevIndex(staleSlot, len);
     (e = tab[i]) != null;
     i = prevIndex(i, len)){

    if (e.get() == null){
        slotToExpunge = i;
    }
}
```

接着开始从`staleSlot`向后查找，也是碰到`Entry`为`null`的桶结束（其实这两个下标就相当于给了一个清理的区间）。 如果迭代过程中，**碰到 k == key**，这说明这里是替换逻辑，替换新数据并且交换当前`staleSlot`位置。如果`slotToExpunge == staleSlot`，这说明`replaceStaleEntry()`一开始向前查找过期数据时并未找到过期的`Entry`数据，接着向后查找过程中也未发现过期数据，修改开始探测式清理过期数据的下标为当前循环的 index，即`slotToExpunge = i`。最后调用`cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);`进行启发式过期数据清理。

```java
if (k == key) {
    e.value = value;

    tab[i] = tab[staleSlot];
    tab[staleSlot] = e;
	// 表明一开始向前查找数据并未找到过期的Entry，于是更新slotToExpunge为当前位置
    if (slotToExpunge == staleSlot)
        slotToExpunge = i;

    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    return;
}
```

`cleanSomeSlots()`和`expungeStaleEntry()`方法后面都会细讲，这两个是和清理相关的方法，一个是过期`key`相关`Entry`的启发式清理(`Heuristically scan`)，另一个是过期`key`相关`Entry`的探测式清理。

```java
// 往后迭代的过程中如果没有找到k == key的数据，且碰到Entry为null的数据，则结束当前的迭代操作。此时说明这里是一个添加的逻辑，将新的数据添加到table[staleSlot] 对应的slot中。
tab[staleSlot].value = null;
tab[staleSlot] = new Entry(key, value);
```

最后判断除了`staleSlot`以外，还发现了其他过期的`slot`数据，就要开启清理数据的逻辑：

```java
if (slotToExpunge != staleSlot)
 	cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
```

### `ThreadLocalMap`过期 key 的探测式清理流程

`ThreadLocalMap`对过期`key`的清理方式有两种：探测式清理和启发式清理。

#### 探测式清理

探测式清理执行`expungeStaleEntry`方法，遍历散列数组，从开始位置向后清理过期的数据，将过期数据的`Entry`设置为`null`，沿途中碰到为过期的数据则将此数据`rehash`后重新在`table`中定位，如果定位的位置已经存在数据，则会将未过期的数据放到靠近此位置的`Entry`为`null`的桶中，使得`rehash`之后的数据的`Entry`位置距离正确的桶的位置更近一些。具体如下图所示：

![image-20220316214731956](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316214731956.png)

`set(27)`经过哈希计算之后应该放在`index=4`的位置，由于`index=4`的位置已经有数据，所以需要向后遍历最终将数据放在`index=7`的位置，放入一段时间后`index=5`中的`Entry`中数据`key`为null。

![image-20220316214948098](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316214948098.png)

如果此时有其他数据需要放入到`map`中，则会出发探测式清理操作。如上图所示，执行探测式清理后，`index=5`的数据被清理掉，继续往后迭代，到了`index=7`的位置时，经过`rehash`操作发现该元素正确的位置应该为`index=4`，但是这个位置已经存在数据，向后查找离`index=4`最近的`Entry=null`的节点（刚刚被清理掉`index=5`），于是就将`index=7`的数据移动到`index=5`的位置中。

经过一轮探测式清理后，`key`过期的数据会被清理掉，没过期的数据经过`rehash`重定位后所处的桶位置理论上更接近`i= key.hashCode & (tab.len - 1)`的位置。这种优化会提高整个散列表查询性能。

接着看下`expungeStaleEntry()`具体流程，先通过原理图来介绍其大致流程，假设`expungeStaleEntry(3)` 来调用此方法，如下图所示，可以看到`ThreadLocalMap`中`table`的数据情况，接着执行清理操作：

![image-20220316215459074](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316215459074.png)

第一步是清空当前`staleSlot`位置的数据，`index=3`位置的`Entry`变成了`null`。然后接着往后探测：

![image-20220316215546675](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316215546675.png)

执行完第二步后，如性爱图所示，`index=4 `的元素挪到` index=3 `的槽位中。

继续往后迭代检查，碰到正常数据，计算该数据位置是否偏移，如果被偏移，则重新计算`slot`位置，目的是让正常数据尽可能存放在正确位置或离正确位置更近的位置。

![image-20220316215628481](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316215628481.png)

在往后迭代的过程中碰到空的槽位，终止探测，这样一轮探测式清理工作就完成了，其具体**实现源代码**如下：

```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

这里还是以`staleSlot=3` 来做示例说明，首先是将`tab[staleSlot]`槽位的数据清空，然后设置`size--` 接着以`staleSlot`位置往后迭代，如果遇到`k==null`的过期数据，也是清空该槽位数据，然后`size--`

```java
ThreadLocal<?> k = e.get();
if (k == null) {
    e.value = null;
    tab[i] = null;
    size--;
}
```

如果`key`没有过期，重新计算当前`key`的下标位置是不是当前槽位下标位置，如果不是，那么说明产生了`hash`冲突，此时以新计算出来正确的槽位位置往后迭代，找到最近一个可以存放`entry`的位置。

```java
// 计算hashcode 判断其是否是当前位置
int h = k.threadLocalHashCode & (len - 1);
if (h != i) {
    tab[i] = null;
	// 找到离h最近的位置存放entry
    while (tab[h] != null)
        h = nextIndex(h, len);

    tab[h] = e;
}
```

这里是处理正常的产生`Hash`冲突的数据，经过迭代后，有过`Hash`冲突数据的`Entry`位置会更靠近正确位置，这样可以提高查询时候的效率。

### `ThreadLocalMap`扩容机制

在`ThreadLocalMap.set()`方法的最后，如果执行完启发式清理工作后，未清理到任何数据，且当前散列数组中`Entry`的数量已经达到了列表的扩容阈值`(len*2/3)`，就开始执行`rehash()`逻辑：

```java
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```

以下是`rehash()`的具体实现：

```java
private void rehash() {
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

这里首先会执行探测式清理操作，从`table`的起始位置向后遍历进行清理。清理完成之后，再来通过判断`size >= threshold - threshold / 4`来决定是否进行扩容操作。

而上文讲到的`rehash()`操作是在`size>=threshold`，也就是先判断`size>=threshold`以此来决定是否执行`rehash`操作，`rehash()`操作会继续探测式清理工作，清理完成之后再判断`size >= threshold - threshold / 4`以此来觉得是否进行`resize()`操作，如下图所示：

![image-20220316223129327](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316223129327.png)

接下来看一下`resize()`方法，其源码如下：

扩容后的`tab`的大小为`oldLen*2`，然后会遍历旧的散列表，重新哈希计算元素的位置，然后将其放到新的`tab`数组中，如果出现`hash`冲突则会往后寻找最近的`entry`为`null`的位置，遍历完成之后，`oldTab`中所有`entry`数据都已经放到`newTab`中。重新计算下次扩容的阈值`setThreshold(newLen)`。

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null;
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### `ThreadLocalMap.get()`详解

首先通过图来展示`get()`方法的流程，大致分为两种情况：

1. 通过查找`key`值计算出其在散列表中的位置，然后如果该位置中的`Entry`的`key`和查找的`key`一致，那么直接将`value`返回

   ![image-20220316223930111](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316223930111.png)

2. 通过查找`key`值计算出其在散列表中的位置，但是该位置中的`Entry`的`key`和查找的`key`不一致，如下图所示：

   ![image-20220316224030814](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316224030814.png)

   以`get(ThreadLocal1)`为例，通过`hash`计算后，正确的`slot`位置应该是 4，而`index=4`的槽位已经有了数据，且`key`值不等于`ThreadLocal1`，所以需要继续往后迭代查找。

   迭代到`index=5`的数据时，此时`Entry`的`key=null`，触发一次探测式数据回收操作，执行`expungeStaleEntry()`方法，执行完后，`index 5,8`的数据都会被回收，而`index 6,7`的数据都会前移，此时继续往后迭代，到`index = 6`的时候即找到了`key`值相等的`Entry`数据，如下图所示：

   ![image-20220316224153853](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316224153853.png)

   其源码如下：

   ```java
   private Entry getEntry(ThreadLocal<?> key) {
       int i = key.threadLocalHashCode & (table.length - 1);
       Entry e = table[i];
       if (e != null && e.get() == key)
           return e;
       else
           return getEntryAfterMiss(key, i, e);
   }
   
   private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
       Entry[] tab = table;
       int len = tab.length;
   
       while (e != null) {
           ThreadLocal<?> k = e.get();
           if (k == key)
               return e;
           if (k == null)
               expungeStaleEntry(i);
           else
               i = nextIndex(i, len);
           e = tab[i];
       }
       return null;
   }
   ```

### `ThreadLocalMap`过期key的启发式清理流程

上面介绍的都是`ThreadLocalMap`过期key的探测式清理，接下来介绍**启发式清理流程（Heuristically scan some cells looking for stale entries.）**如下图所示：

![image-20220316224403514](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/image-20220316224403514.png)

其源码如下：

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

### `InheritableThreadLocal`

使用`ThreadLocal`的时候，在异步场景下是无法给子线程共享父线程中创建的线程副本数据的。为了解决这个问题，JDK 中还有一个`InheritableThreadLocal`类：

```java
public class InheritableThreadLocalDemo {
    public static void main(String[] args) {
        ThreadLocal<String> ThreadLocal = new ThreadLocal<>();
        ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
        ThreadLocal.set("父类数据:threadLocal");
        inheritableThreadLocal.set("父类数据:inheritableThreadLocal");

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("子线程获取父类ThreadLocal数据：" + ThreadLocal.get());
                System.out.println("子线程获取父类inheritableThreadLocal数据：" + inheritableThreadLocal.get());
            }
        }).start();
    }
}
```

输出如下：

```java
子线程获取父类ThreadLocal数据：null
子线程获取父类inheritableThreadLocal数据：父类数据:inheritableThreadLocal
```

实现原理是子线程是通过在父线程中通过调用`new Thread()`方法来创建子线程，`Thread#init`方法在`Thread`的构造方法中被调用。在`init`方法中拷贝父线程数据到子线程中：

```java
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    this.stackSize = stackSize;
    tid = nextThreadID();
}
```

但`InheritableThreadLocal`仍然有缺陷，一般我们做异步化处理都是使用的线程池，而`InheritableThreadLocal`是在`new Thread`中的`init()`方法给赋值的，而线程池是线程复用的逻辑，所以这里会存在问题。阿里巴巴开源了一个`TransmittableThreadLocal`组件就可以解决这个问题。

### `ThreadLocal`项目中使用实战

#### `ThreadLocal`使用场景

我们现在项目中日志记录用的是`ELK+Logstash`，最后在`Kibana`中进行展示和检索。

现在都是分布式系统统一对外提供服务，项目间调用的关系可以通过 `traceId` 来关联，但是不同项目之间如何传递 `traceId` 呢？

这里我们使用 `org.slf4j.MDC` 来实现此功能，内部就是通过 `ThreadLocal` 来实现的，具体实现如下：

当前端发送请求到**服务 A**时，**服务 A**会生成一个类似`UUID`的`traceId`字符串，将此字符串放入当前线程的`ThreadLocal`中，在调用**服务 B**的时候，将`traceId`写入到请求的`Header`中，**服务 B**在接收请求时会先判断请求的`Header`中是否有`traceId`，如果存在则写入自己线程的`ThreadLocal`中。

![img](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/30.8167dc64.png)

图中的`requestId`即为我们各个系统链路关联的`traceId`，系统间互相调用，通过这个`requestId`即可找到对应链路，这里还有会有一些其他场景：

![img](../../img/ThreadLocal%E8%A7%A3%E6%9E%90/31.742c0f94.png)

针对于这些场景，我们都可以有相应的解决方案，如下所示

#### Feign远程调用解决方案

**服务发送请求：**

```java
@Component
@Slf4j
public class FeignInvokeInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String requestId = MDC.get("requestId");
        if (StringUtils.isNotBlank(requestId)) {
            template.header("requestId", requestId);
        }
    }
}
```

**服务接收请求：**

```java
@Slf4j
@Component
public class LogInterceptor extends HandlerInterceptorAdapter {

    @Override
    public void afterCompletion(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2, Exception arg3) {
        MDC.remove("requestId");
    }

    @Override
    public void postHandle(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2, ModelAndView arg3) {
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String requestId = request.getHeader(BaseConstant.REQUEST_ID_KEY);
        if (StringUtils.isBlank(requestId)) {
            requestId = UUID.randomUUID().toString().replace("-", "");
        }
        MDC.put("requestId", requestId);
        return true;
    }
}
```

#### 线程池异步调用，requestId 传递

因为`MDC`是基于`ThreadLocal`去实现的，异步过程中，子线程并没有办法获取到父线程`ThreadLocal`存储的数据，所以这里可以自定义线程池执行器，修改其中的`run()`方法：

```java
public class MyThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {

    @Override
    public void execute(Runnable runnable) {
        Map<String, String> context = MDC.getCopyOfContextMap();
        super.execute(() -> run(runnable, context));
    }

    @Override
    private void run(Runnable runnable, Map<String, String> context) {
        if (context != null) {
            MDC.setContextMap(context);
        }
        try {
            runnable.run();
        } finally {
            MDC.remove();
        }
    }
}
```

#### 使用 MQ 发送消息给第三方系统

在 MQ 发送的消息体中自定义属性`requestId`，接收方消费消息后，自己解析`requestId`使用即可。
