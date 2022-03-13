### Java常见并发容器总结

JDK提供的并发容器大部分都在`java.util.concurrent(juc)`包

- `ConcurentHahsMap`：线程安全的`HashMap`
- `CopyOnWriteArrayList`：线程安全的List，适用于读多写少的场景。
- `ConcurrentLinkedQueue`：高效的并发队列，使用链表实现。可以当作一个线程安全的`LinkedList`，这是一个非阻塞队列。
- `BlockingQueue`：这个一个接口，JDK内部通过链表、数组等方式实现了这个接口。表示阻塞队列，适合用于作为数据共享的通道。
- `ConcurrentSkipListMap`：线程安全的跳表。

#### `ConcurrentHashMap`

​	首先`HashMap`不是线程安全 的，在并发场景下如果要保证一种可行的方式是使用`Collections.synchronizedMap()`方法来包装`HashMap`。但这是通过使用一个全局锁来做到并发控制，这对性能有很大的影响。

​	于是`ConcurrentHashMap`就出现了。在`ConcurrentHashMap`中，无论是读操作还是写操作都能保证很高的性能；在读操作的时候（几乎）不需要加锁，而在写操作时通过锁分段技术只对所操作的段加锁而不影响客户端对其他段的访问。

#### `CopyOnWriteArrayList`

```java
public class CopyOnWriteArrayList<E>
extends Object
implements List<E>, RandomAccess, Cloneable, Serializable
```

​	在读多写少的场景中，由于读操作根本不会修改原有的数据，因此对于每次读取都进行加锁会影响性能。所有应该允许多个线程同时访问`List`的内部数据，因为读操作是线程安全的。这与`ReentrantReadWriteLock`读写锁的思想类似，就是读操作见共享，写操作与读操作或者写操作互斥。但是JDK中提供了`CopyOnWriteArrayList`类比相比于读写锁的思想又更进了一步。为了将读取的性能发挥到极致，`CopyOnWriteArrayList`读取是完全不用加锁的，并且写入操作不会阻塞读取操作。只有写入和写入之间需要进行同步等待。

​	**如何做到？**

​	`CopyOnWriteArrayList`类的所有可变操作（add,set等等）都是通过创建底层数组的副本来实现的。当List需要被修改的时候，并不会修改原有内容，而是对原有数据进行一次复制，将修改的内容写入副本。写完之后，再将修改完的副本替换原来的数据，这样就可以保证写操作不会影响读操作了。所谓`CopyOnWrite`也就是说：在计算机中，如果需要对一块内存区域进行修改，不在原有的内存块中进行写操作，而是将其拷贝一份在新的内存区域中，在新的内存区域中进行写操作，写完之后，就将指向原来内存的指针指向新的内存，原来的内存就可以被回收掉了。

##### `CopyOnWriteArrayList`读取和写入源码分析

`CopyOnWriteArrayList`读取操作的实现

读取操作没有任何同步控制和锁操作，理由就是内部数组不会发生修改，只会被另一个数组替换。因此可以保证数据安全。

```java
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
    public E get(int index) {
        return get(getArray(), index);
    }
    @SuppressWarnings("unchecked")
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
    final Object[] getArray() {
        return array;
    }
```

`CopyOnWriteArrayList`写入操作的实现

`CopyOnWriteArrayList`写入操作`add()`方法在添加集合的时候加了锁，保证了同步，避免了多线程写的时候会copy出来多个副本。

```java
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();//加锁
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);//拷贝新数组
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();//释放锁
        }
    }
```

#### `ConcurrentLinkedQueue`

​	Java提供的线程安全的`Queue`可以分为阻塞队列和非阻塞队列，其中阻塞队列的典型例子是`BlockingQueue`，非阻塞队列的典型例子是`ConcurrentLinkedQueue`，在实际应用中要根据实际需要选用阻塞队列或者非阻塞队列。阻塞队列可以通过加锁来实现，非阻塞队列可以通过CAS操作来实现。

​	`ConcurrentLinkedQueue`使用队列作为其数据结构。`ConcurrentLinkedQueue`在高并发环境中拥有极佳的性能是因为其内部复杂的实现。其主要使用CAS非阻塞算法来实现线程安全。适用于对性能要求相对较高，同时对队列的读写存在多个线程同时进行的场景，即如果队列加锁的成本较高则适合用无锁的`ConcurrentLinkedQueue`来替代。

#### `BlockingQueue`

​	`BlockingQueue`是阻塞队列。阻塞队列被广泛使用在“生产者-消费者”问题中，其原因是`BlockingQueue`提供了可阻塞插入和移除的方法。当队列容器满，生产者进程会被阻塞；当队列容器为空时，消费者线程会被阻塞，直到队列非空时为止。

​	`BlockingQueue`是一个接口，继承自`Queue`，所有其实现类也可作为`Queue`的实现来使用。而`Queue`又继承自`Collection`接口。下面是`BlockingQueue`的相关实现类。

![image-20220304204440882](../../img/Java%E5%B8%B8%E8%A7%81%E5%B9%B6%E5%8F%91%E5%AE%B9%E5%99%A8%E6%80%BB%E7%BB%93/image-20220304204440882.png)

​	下面主要介绍三个常见的`BlockingQueue`的实现类：`ArrayBlockingQueue`、`LinkedBlockingQueue`、`PriorityBlockingQueue`。

##### 	**`ArrayBlockingQueue`**

​	`ArrayBlockingQueue`是`BlockingQueue`接口的有界队列实现类，底层采用数组来实现。

```java
public class ArrayBlockingQueue<E>
extends AbstractQueue<E>
implements BlockingQueue<E>, Serializable{}
```

​	`ArrayBlockingQueue`一旦创建，容量不能改变，其并发控制采用可重入锁`ReentrantLock`，不管是插入操作还是读操作，都需要获取到锁才能进行操作。当队列容量满时，尝试将元素加入队列将导致操作阻塞；尝试从一个空队列中取一个元素也会同样阻塞。

​	`ArrayBlockingQueue`默认情况下不能保证线程访问队列的公平性，所谓公平性是指严格按照线程等待的绝对时间顺序，即最先等待的线程能够最先访问到`ArrayBlockingQueue`。而非公平性则是指访问`ArrayBlockingQueue`的顺序不是遵守严格的时间顺序，有可能存在，当`ArrayBlockingQueue`可以被访问时，长时间阻塞的线程依然无妨访问到`ArrayBlockingQueue`。如果保证公平性，通常会降低吞吐量。如果需要获得公平性的`ArrayBlockingQueue`，可采用如下代码：

```java
private static ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(10,true);
```

##### 	`LinkedBlockingQueue`

​	`LinkedBlockingQueue`底层基于**单向链表**实现的阻塞队列，可以当作无界队列也可以当作有界队列来使用，同样满足FIFO的特性，与`ArrayBlockingQueue`相比起来具有更高的吞吐量，为了防止`LinkedBlockingQueue`容量迅速增。损耗大量内存。通常在创建`LinkedBlockingQueue`对象时，会 指定其大小，如果未指定，容量等于`Integer.MAX_VALUE`。其构造方法如下：

```java
    /**
     *某种意义上的无界队列
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    /**
     *有界队列
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
```

##### 	`PriorityBlockingQueue`

​	`PriorityBlockingQueue`是一个支持优先级的无界阻塞队列。默认情况下元素采用自然顺序进行排序，也可以通过自己定义类实现`compareTo()`方法来指定元素排序规则，或者初始化时通过构造器参数`Comparator`来指定排序规则。

​	`PriorityBlockingQueue`并发控制采用的是可重入锁`ReentranLock`，队列为无界队列（`ArrayBlockingQueue`是有界队列，`LinkedBlockingQueue`也可以通过构造函数中传入`capaciy`指定队列的最大容量，但是`PriorityBlockingQueue` 只能指定初始的队列大小，后面插入元素的时候，**如果空间不够的话会自动扩容**）。

​	`PriorityBlockingQueue`就是`PriorityQueue`的线程安全版本。不可以插入`null`值，同时，插入队列的对象必须是可比较大小的，否则会抛出`ClassCastException`异常。其`put`方法不会阻塞，因为它是无界队列（`take`方法在队列为空的时候会阻塞）。

#### `ConcurrentSkipListMap`

​	什么是跳表？对于一个单链表，即使链表是有序的，如果我们想要在其中查找某个数据，也只能从头到尾遍历链表，这样效率低下。跳表（跳跃列表）是一种可以用来快速查询，插入和删除的有序的数据链表。跳表的平均查找和插入时间复杂度都是O(logn)。在高并发场景下，只需要部分锁即可保证线程安全。

![image-20220305213026101](../../img/Java%E5%B8%B8%E8%A7%81%E5%B9%B6%E5%8F%91%E5%AE%B9%E5%99%A8%E6%80%BB%E7%BB%93/image-20220305213026101.png)

​	跳表通过维护一个多层次的链表，最低层维护了跳表所有的元素，且每一层链表中的元素是前一层链表元素的子集。跳表内的所有链表的元素都是有序的。查找时，可以从顶层链表开始找。一旦发现被查找的元素大于当前链表中的取值，就会转入下一层了链表继续查找。整个查找的过程是“跳跃”的，显然，跳表是一种利用空间换时间的策略。

​	使用跳表实现`Map`和使用哈希算法实现`Map`的另外一个不同之处是：哈希并不会保存元素的顺序，而跳表的所有元素都是有序的。因此在对跳表进行遍历时，得到的结果是有序的。因此，需要元素有序，跳表可以成为候选项之一。
