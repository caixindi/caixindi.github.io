---
title: Atomic原子类总结
date: 2022-03-12
categories:
- Java
tags:
- Java并发编程
language: zh-CN
toc: true
---



### Atomic原子类

`Atomic`是基于`unsafe`类和自旋操作实现的，要理解`Atomic`首先需要理解CAS。Atomic是指一个操作是不可中断的，即使在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。

所以，所谓源自类就说具有原子/原子操作特征的类。根据操作的数据类型，可以讲JUC包中的原子类分为4类。

> 要理解Atomic首先得了解CAS，**CAS**（Compare and Swap）,其功能就是判断内存中的某个值是否与预期的值相等，相等就用新值更新旧值，否则就不更新。Java中CAS是基于`unsafe`类首先的，所有的`unsafe`类中的方法都是native（原生）方法，直接调用操作系用底层资源执行任务。

**基本类型**

使用原子的方式更新基本类型

- `AtomicInteger`：整形原子类
- `AtomicLong`：长整型原子类
- `AtomicBoolean`：布尔型原子类

**数组类型**

使用原子的方式更新数组里的某个元素

- `AtomicIntegerArray`：整形数组原子类
- `AtomicLongArray`：长整型数组原子类
- `AtomicReferenceArray`：引用类型数组原子类

**引用类型**

- `AtomicReference`：引用类型原子类

- `AtomicMarkabaleReference`：原子更新带有标记的引用类型。该类将boolean标记与引用关联起来

  > AtomicMarkableReference无法解决ABA问题，因为它是将一个boolean值作为是否有效的标记，也就是说它的版本号其实只有两个，修改的时候标记只是在true和false之间切换，这样做并不能解决ABA问题。

- `AtomicStampedReference`：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于可决原子的更新数据和数据版本号，可以解决使用CAS进行原子更新时可能出现的ABA问题

  > CAS ABA问题：第一个线程取到了变量x的值A，然后做一些其他的操作，在此期间第二个线程也取到了变量x的值A，然后第二个线程把变量x的值改为B，最好又把变量x的值改为A，最后第一个线程对变量x进行操作，但这个时候虽然x的值还是A，`compareAndSet`操作依然成功。也就是第一个线程对第二个线程的操作是没有感知的。这在某些场景下是会产生问题的。比如在银行系统中，如果无法感知第二个线程，这是很危险的。

**对象的属性修改类型**

- `AtomicIntegerFieldUpdater`：原子更新整型字段的更新器
- `AtomicLongFieldUpdater`：原子更新长整型字段的更新器
- `AtomicReferenceFieldUpdater`：原子更新引用类型里的字段

### 基本类型原子类

以`AtomicInteger`为例，其常用类方法如下：

```java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）
public final void lazySet(int newValue)//最终设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

使用示例：

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerDemo {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		int temvalue = 0;
		AtomicInteger i = new AtomicInteger(0);
		temvalue = i.getAndSet(3);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);//temvalue:0;  i:3
		temvalue = i.getAndIncrement();
		System.out.println("temvalue:" + temvalue + ";  i:" + i);//temvalue:3;  i:4
		temvalue = i.getAndAdd(5);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);//temvalue:4;  i:9
	}

}
```

#### 基本数据类型原子类的优势

1. **多线程环境不使用原子类保证线程安全**

   需要使用**synchronized**，示例如下：

   ```java
   class Test {
           private volatile int count = 0;
           //若要线程安全执行执行count++，需要加锁
           public synchronized void increment() {
                     count++;
           }
   
           public int getCount() {
                     return count;
           }
   }
   ```

2. **多线程环境使用原子类保证线程安全（基本数据类型）**

   ```java
   class Test2 {
           private AtomicInteger count = new AtomicInteger();
   
           public void increment() {
                     count.incrementAndGet();
           }
         //使用AtomicInteger之后，不需要加锁，也可以实现线程安全。
          public int getCount() {
                   return count.get();
           }
   }
   ```

#### AtomicInteger线程安全原理分析

部分源码如下：

```java
    // setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

AtomicInteger类主要利用CAS+volatile和native方法来保证原子操作，从而避免使用synchronized的高开销，执行效率大为提升。

CAS就是拿期望值和原来的值进行比较，如果相同就更新，如果不相同就不更新。Unsafe类的`objectFieldOffset()`方法是一个native方法，这个方法是用来拿到旧值的内存地址，另外value是一个volatile变量，在内存中可见，因此JVM可以保证在任何时刻任何线程总能拿到该变量的最新值。

### 数组类型原子类

#### 数组类型原子类介绍

使用原子的方式更新数组里的某个元素

- `AtomicIntegerArray`：整形数组原子类
- `AtomicLongArray`：长整型数组原子类
- `AtomicReferenceArray`：引用数组类型原子类

上面三个类提供的方法几乎相同，这边以`AtomicIntegerArray`为例进行介绍。

`AtomicIntegerArray`类常用方法

```java
public final int get(int i) //获取 index=i 位置元素的值
public final int getAndSet(int i, int newValue)//返回 index=i 位置的当前的值，并将其设置为新值：newValue
public final int getAndIncrement(int i)//获取 index=i 位置元素的值，并让该位置的元素自增
public final int getAndDecrement(int i) //获取 index=i 位置元素的值，并让该位置的元素自减
public final int getAndAdd(int i, int delta) //获取 index=i 位置元素的值，并加上预期的值
boolean compareAndSet(int i, int expect, int update) //如果输入的数值等于预期值，则以原子方式将 index=i 位置的元素值设置为输入值（update）
public final void lazySet(int i, int newValue)//最终 将index=i 位置的元素设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

`AtomicIntegerArray`常见方法的使用

```java

import java.util.concurrent.atomic.AtomicIntegerArray;

public class AtomicIntegerArrayTest {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		int temvalue = 0;
		int[] nums = { 1, 2, 3, 4, 5, 6 };
		AtomicIntegerArray i = new AtomicIntegerArray(nums);
		for (int j = 0; j < nums.length; j++) {
			System.out.println(i.get(j));
		}
		temvalue = i.getAndSet(0, 2);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);
		temvalue = i.getAndIncrement(0);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);
		temvalue = i.getAndAdd(0, 5);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);
	}

}
```

### 引用类型原子类

#### 引用类型原子类介绍

基本类型原子类只能更新一个变量，如果需要原子更新多个变量，需要使用引用类型原子类。

- `AtomicReference`：引用类型原子类
- `AtomicStampedReference`：原子更新带有版本号的引用类型。该类将整数值与引用类型关联起来，可用于解决原子更新数据和数据的版本后，可以解决使用CAS进行原子更新时可能出现的ABA问题。
- `AtomicMarkableReference`：原子更新带有标记的引用类型。该类将boolean标记与引用类型关联起来。

上面三个类可以提供的方法几乎相同，接下来以`AtomicReference`为例介绍：

```java
import java.util.concurrent.atomic.AtomicReference;

public class AtomicReferenceTest {
	public static void main(String[] args) {
		AtomicReference<Person> ar = new AtomicReference<Person>();
		Person person = new Person("SnailClimb", 22);
		ar.set(person);
		Person updatePerson = new Person("Daisy", 20);
		ar.compareAndSet(person, updatePerson);

		System.out.println(ar.get().getName());
		System.out.println(ar.get().getAge());
	}
}

class Person {
	private String name;
	private int age;

	public Person(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

}
```

结果如下：

```java
Daisy
20
```

上诉代码首先创建了一个Person对象，然后把Person对象设置进`AtomicReference`对象中，然后调用compareAndSet方法，该方法就是通过CAS操作设置ar。如果ar的值为person的话，则将其设置为updatePerson。实现原理与`AtomicInteger`类中的compareAndSet方法相同。

#### `AtomicStampedReference`类使用示例

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class AtomicStampedReferenceDemo {
    public static void main(String[] args) {
        // 实例化、取当前值和 stamp 值
        final Integer initialRef = 0, initialStamp = 0;
        final AtomicStampedReference<Integer> asr = new AtomicStampedReference<>(initialRef, initialStamp);
        System.out.println("currentValue=" + asr.getReference() + ", currentStamp=" + asr.getStamp());

        // compare and set
        final Integer newReference = 666, newStamp = 999;
        final boolean casResult = asr.compareAndSet(initialRef, newReference, initialStamp, newStamp);
        System.out.println("currentValue=" + asr.getReference()
                + ", currentStamp=" + asr.getStamp()
                + ", casResult=" + casResult);

        // 获取当前的值和当前的 stamp 值
        int[] arr = new int[1];
        final Integer currentValue = asr.get(arr);
        final int currentStamp = arr[0];
        System.out.println("currentValue=" + currentValue + ", currentStamp=" + currentStamp);

        // 单独设置 stamp 值
        final boolean attemptStampResult = asr.attemptStamp(newReference, 88);
        System.out.println("currentValue=" + asr.getReference()
                + ", currentStamp=" + asr.getStamp()
                + ", attemptStampResult=" + attemptStampResult);

        // 重新设置当前值和 stamp 值
        asr.set(initialRef, initialStamp);
        System.out.println("currentValue=" + asr.getReference() + ", currentStamp=" + asr.getStamp());

        // [不推荐使用，除非搞清楚注释的意思了] weak compare and set
        // 困惑！weakCompareAndSet 这个方法最终还是调用 compareAndSet 方法。[版本: jdk-8u191]
        // 但是注释上写着 "May fail spuriously and does not provide ordering guarantees,
        // so is only rarely an appropriate alternative to compareAndSet."
        // todo 感觉有可能是 jvm 通过方法名在 native 方法里面做了转发
        final boolean wCasResult = asr.weakCompareAndSet(initialRef, newReference, initialStamp, newStamp);
        System.out.println("currentValue=" + asr.getReference()
                + ", currentStamp=" + asr.getStamp()
                + ", wCasResult=" + wCasResult);
    }
}
```

输出结果如下：

```java
currentValue=0, currentStamp=0
currentValue=666, currentStamp=999, casResult=true
currentValue=666, currentStamp=999
currentValue=666, currentStamp=88, attemptStampResult=true
currentValue=0, currentStamp=0
currentValue=666, currentStamp=999, wCasResult=true
```

#### `AtomicMarkableReference`类使用示例

```java
import java.util.concurrent.atomic.AtomicMarkableReference;

public class AtomicMarkableReferenceDemo {
    public static void main(String[] args) {
        // 实例化、取当前值和 mark 值
        final Boolean initialRef = null, initialMark = false;
        final AtomicMarkableReference<Boolean> amr = new AtomicMarkableReference<>(initialRef, initialMark);
        System.out.println("currentValue=" + amr.getReference() + ", currentMark=" + amr.isMarked());

        // compare and set
        final Boolean newReference1 = true, newMark1 = true;
        final boolean casResult = amr.compareAndSet(initialRef, newReference1, initialMark, newMark1);
        System.out.println("currentValue=" + amr.getReference()
                + ", currentMark=" + amr.isMarked()
                + ", casResult=" + casResult);

        // 获取当前的值和当前的 mark 值
        boolean[] arr = new boolean[1];
        final Boolean currentValue = amr.get(arr);
        final boolean currentMark = arr[0];
        System.out.println("currentValue=" + currentValue + ", currentMark=" + currentMark);

        // 单独设置 mark 值
        final boolean attemptMarkResult = amr.attemptMark(newReference1, false);
        System.out.println("currentValue=" + amr.getReference()
                + ", currentMark=" + amr.isMarked()
                + ", attemptMarkResult=" + attemptMarkResult);

        // 重新设置当前值和 mark 值
        amr.set(initialRef, initialMark);
        System.out.println("currentValue=" + amr.getReference() + ", currentMark=" + amr.isMarked());

        // [不推荐使用，除非搞清楚注释的意思了] weak compare and set
        // 困惑！weakCompareAndSet 这个方法最终还是调用 compareAndSet 方法。[版本: jdk-8u191]
        // 但是注释上写着 "May fail spuriously and does not provide ordering guarantees,
        // so is only rarely an appropriate alternative to compareAndSet."
        // todo 感觉有可能是 jvm 通过方法名在 native 方法里面做了转发
        final boolean wCasResult = amr.weakCompareAndSet(initialRef, newReference1, initialMark, newMark1);
        System.out.println("currentValue=" + amr.getReference()
                + ", currentMark=" + amr.isMarked()
                + ", wCasResult=" + wCasResult);
    }
}
```

输出结果如下：

```java
currentValue=null, currentMark=false
currentValue=true, currentMark=true, casResult=true
currentValue=true, currentMark=true
currentValue=true, currentMark=false, attemptMarkResult=true
currentValue=null, currentMark=false
currentValue=true, currentMark=true, wCasResult=true
```

### 对象的属性修改类型原子类

如果需要原子更新某个类的某个字段时，需要用到对象的属性修改类型原子类。

- `AtomicIntegerFieldUpdater`：原子更新整形字段的更新器
- `AtomicLongFieldUpdater`：原子更新长整形字段的更新器
- `AtomicReferenceFieldUpdater`：原子更新引用类型里的字段的更新器

原子地更新对象的属性需要两步。第一步，因为对象的属性修改类型原子类都是抽象类，所以每次使用都必须使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性；第二步，更新的对象属性必须使用public volatile修饰符。

上面三个类提供的方法几乎相同，所以我们这里以`AtomicIntegerFieldUpdater`为例子来介绍。

`AtomicIntegerFieldUpdater`类示例代码：

```java
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

public class AtomicIntegerFieldUpdaterTest {
	public static void main(String[] args) {
		AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater.newUpdater(User.class, "age");

		User user = new User("Java", 22);
		System.out.println(a.getAndIncrement(user));// 22
		System.out.println(a.get(user));// 23
	}
}

class User {
	private String name;
	public volatile int age;

	public User(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

}
```

输出结果：

```java
22
23
```

### CAS与synchronized比较

CAS支持多个线程并发修改，并发程度高，而synchronized一次只有一个线程修改，并发程度低；

CAS只支持一个共享变量的原子操作；

synchronized可以对多个变量进行加锁；

CAS会出现ABA问题（可以解决）；

