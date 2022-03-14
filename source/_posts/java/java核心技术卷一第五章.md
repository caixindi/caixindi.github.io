---
title: Java核心技术卷一-第五章
date: 2021-12-03
categories:
- Java
tags:
- Java知识点
language: zh-CN
toc: true
---

#### [getClass()](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/lang/Object.html#getClass()) 

返回运行时类的一个对象

```java
import java.util.GregorianCalendar;
public class ObjectDemo {
   public static void main(String[] args) {
      // create a new ObjectDemo object
      GregorianCalendar cal = new GregorianCalendar();
      // print current time
      System.out.println("" + cal.getTime());
      // print the class of cal
      System.out.println("" + cal.getClass());
      // create a new Integer
      Integer i = new Integer(5);
      // print i
      System.out.println("" + i);
      // print the class of i
      System.out.println("" + i.getClass());
   }
}
```

<!--more-->

执行结果：

```
Mon Oct 11 09:30:37 UTC 2021
class java.util.GregorianCalendar
5
class java.lang.Integer
```

#### ArrayList

数组列表管理着对象引用的一个内部数组，可以使用add方法向其中添加新的对象，当内部数组满了的时候，数组列表会自动创建一个更大的数组，并且将所有的对象从较小数组拷贝到较大数组中去。可以调用ensureCapacity(int minCapacity)方法来设置数组中可能或者已经确定要存储的对象数，这样将会给数组列表分配一个指定大小minCapacity的内部数组(只是一个潜力，还可以扩容)，这样在之后调用add方法的时候不需要重新分配空间，因为数组扩容的非常耗费时间和内存。也可以在初始化的时候将初始容量直接传递给构造器，比如ArrayList<Person> arrayList =new ArrayList<>(100)，这个方法也可以用来进行扩容。

**注意数组列表和数组的区别**

```java
ArrayList<Person> arrayList =new ArrayList<>();
arrayList.ensureCapacity(100);
/*这个时候只是赋予数组列表存储100个对象的潜力,也就是底层数组的长度
这个size和数组的length不同，ArrayList的length就是底层数组长度，但这个长度
对用户的意义不大，一般用户所需要的都是当前实际存储的对象数，因此ArrayList中一个私有成员变量size
可以通过size()方法获得，它返回了数组列表中实际存储的对象数
*/
System.out.println(arrayList.size());
Person[] people=new Person[100];
//这是真的给数组分配了100个元素的存储空间，有100个空位置可以用
System.out.println(people.length);
/*输出
0
100
*/
```

> 初始化容量对性能的影响

如果用户在构造一个ArrayList的时候**没有指定容量大小**，那么将会调用ArrayList的无参构造函数，默认容量为10。接下来看ArrayList的无参构造函数：

```java
/**
 * Constructs an empty list with an initial capacity of ten.
 */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

事实上在构造完成的时候，它的底层数组是空的，并不是10，只有在调用了add方法之后，才会将底层数组扩容到10，看源码：

```java
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    } 

	/**
     * This helper method split out from add(E) to keep method
     * bytecode size under 35 (the -XX:MaxInlineSize default value),
     * which helps when add(E) is called in a C1-compiled loop.
     */
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        size = s + 1;
    }
```

add方法就是先比较ArrayList的实际大小和底层数组的长度，如果已经相等则需要扩容，调用grow方法，来看grow方法：

这边包括上边的代码有一个需要注意的地方就是**size在构造函数中并没有赋值，但其实它有一个初始值为0，虽然在定义成员变量的时候没有给出，但编译器会给它一个初始值，int类型的变量初值为0，但是在方法体内的局部变量如果不赋予初值，则会编译报错**。

```java
    private Object[] grow() {
        return grow(size + 1);
    }
    
    
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     * @throws OutOfMemoryError if minCapacity is less than zero
     */
    private Object[] grow(int minCapacity) {
        //记录当前底层数组的长度
        int oldCapacity = elementData.length;
        //如果底层数组的长度大于0或者是底层数组并不是空数组，那么就执行扩容操作
        if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            //将newCapacity设置为minCapacity和1.5*oldCapacity中的最大值
            int newCapacity = ArraysSupport.newLength(oldCapacity,
                    minCapacity - oldCapacity, /* minimum growth */
                    oldCapacity >> 1           /* preferred growth */);
            //生成了新的数组对象，将原数组中的内容拷贝到新的数组中
            return elementData = Arrays.copyOf(elementData, newCapacity);
        } else {
            //底层数组长度为小于等于0或者底层数组为空，那么就将底层数组变成一个容量最小为10的空数组（可以看到下图minCapacity=1，也就是初始化之后如果调用add方法，是会将arraylist扩容为10）
            return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
        }
    }
```

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20211014160714666.png)

但是如果在初始化的时候给了容量大小，那么会执行有参构造函数，如下，这个时候会生成一个大小为给定容量的数组，将其引用地址赋给elementData，这个时候调用add方法，在实际大小不超过底层数组大小的时候就不需要进行扩容。

```java
    /**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

于是现在设想这么一种情况，如果不给的初始容量，并且后面也不调用ensureCapacity方法，那么在实际容量(调用add方法)达到（0->10->15->22->33->49->73）的时候都会进行扩容操作，如果最终的元素个数为100个，那么总计执行了7次扩容（grow）操作，但如果在一开始就给定了初始容量100，将会直接给底层数组将直接变成长度为100的数组，也就相当于只执行了一次扩容操作。并且这两种“扩容“操作的代价并不是一样的，前者的扩容操作在后面的阶段还包括了数据的移动，所以是相当耗费时间和内存的，如果在数据量比较大的情况下，结果就可想而知了。

```java
    public void ensureCapacity(int minCapacity) {
        //初始化容量必须大于底层数组的长度并且m底层数组不为空或者初始化容量大于默认容量10
        if (minCapacity > elementData.length
            && !(elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
                 && minCapacity <= DEFAULT_CAPACITY)) {
            modCount++;
            grow(minCapacity);
        }
    }

 /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     * @throws OutOfMemoryError if minCapacity is less than zero
     */
//真正实现扩容的函数
    private Object[] grow(int minCapacity) {
        //获得当前的实际容量
        int oldCapacity = elementData.length;
        if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            //将newCapacity设置为minCapacity和1.5*oldCapacity中的最大值
            int newCapacity = ArraysSupport.newLength(oldCapacity,
                    minCapacity - oldCapacity, /* minimum growth */
                    oldCapacity >> 1           /* preferred growth */);
            //生成了新的数组对象，将原数组中的内容拷贝到新的数组中
            return elementData = Arrays.copyOf(elementData, newCapacity);
        } else {
            //初始化为默认容量
            return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
        }
    }
```