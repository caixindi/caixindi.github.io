> 什么是hash

![image-20211012163401291](D:/Typora-note/img/hashcode%E7%9A%84%E7%90%86%E8%A7%A3/image-20211012163401291.png)

​	hash其实就是一个函数，其中实现了计算hash值的方法，用来得到hash值的算法有许多种，常见的就有:直接取余法，乘法取整法，平方取中法。

> 位运算符

```wiki
<< : 左移运算符，num << 1,相当于num乘以2  低位补0
>> : 右移运算符，num >> 1,相当于num除以2  高位补0
>>> : 无符号右移，忽略符号位，空位都以0补齐
 % : 模运算 取余
^ :   位异或 第一个操作数的的第n位于第二个操作数的第n位相反，那么结果的第n为也为1，否则为0
 & : 与运算 第一个操作数的的第n位于第二个操作数的第n位如果都是1，那么结果的第n为也为1，否则为0
 | :  或运算 第一个操作数的的第n位于第二个操作数的第n位 只要有一个是1，那么结果的第n为也为1，否则为0
 ~ : 非运算 操作数的第n位为1，那么结果的第n位为0，反之，也就是取反运算（一元操作符：只操作一个数）
```

> hashCode是否存储的是对象地址

显然不是，这是String类下的hashCode方法

```java
public int hashCode() {
    // The hash or hashIsZero fields are subject to a benign data race,
    // making it crucial to ensure that any observable result of the
    // calculation in this method stays correct under any possible read of
    // these fields. Necessary restrictions to allow this to be correct
    // without explicit memory fences or similar concurrency primitives is
    // that we can ever only write to one of these two fields for a given
    // String instance, and that the computation is idempotent and derived
    // from immutable state
    int h = hash;
    if (h == 0 && !hashIsZero) {
        h = isLatin1() ? StringLatin1.hashCode(value)
                       : StringUTF16.hashCode(value);
        if (h == 0) {
            hashIsZero = true;
        } else {
            hash = h;
        }
    }
    return h;
}
//StringUTF16.java
public static int hashCode(byte[] value) {
    int h = 0;
    int length = value.length >> 1;
    for (int i = 0; i < length; i++) {
        h = 31 * h + getChar(value, i);
    }
    return h;
}
//StringLatin1.java
public static int hashCode(byte[] value) {
    int h = 0;
    for (byte v : value) {
        h = 31 * h + (v & 0xff);
    }
    return h;
}
```

这是Integer类下的hashCode方法，直接返回了value的值

![](D:/Typora-note/img/hashcode%E7%9A%84%E7%90%86%E8%A7%A3/image-20211013185430971.png)

这是Float类下的hashCode方法，返回了浮点数IEEE754的表示形式

![image-20211013185952426](D:/Typora-note/img/hashcode%E7%9A%84%E7%90%86%E8%A7%A3/image-20211013185952426.png)

![image-20211013190013169](D:/Typora-note/img/hashcode%E7%9A%84%E7%90%86%E8%A7%A3/image-20211013190013169.png)

```java
//测试
Float f=2.5f;
System.out.println(f.hashCode());
/*输出
1075838976
因为是十进制，转化为32位二进制数
0 10000000 01000000000000000000000
根据IEEE754标准，表示：
1.01*2=10.1B=1*2+1*1/2=2+0.5=2.5D
*/
```

于是可以知道，**hashcode一样的两个对象不一定是一样的，但hashcode不一样的两个对象一定是不一样的**。当两个不同的对象的hashcode相同时，就发生了冲突，这个时候就需要相应的处理冲突方法。

> hashCode方法和equals方法的关系

​	hashcode在什么时候会大有用处呢？Java中的两个容器类分别为Collection和Map，其中的Map和Set不允许存放重复的元素，为了存取效率，它们的底层实现都会用到散列结构。

​	这个时候就会碰到这么一个问题，该如何保证存入的元素不重复，如果使用equals方法，那么就需要依此比较已经存储在map中的所有元素对象，这时候的时间复杂度为O(n)。但有了hashcode在散列表的基础上，一切就要简单许多，在新加入元素的时候，只需要将其哈希值与相应位置进行比较，查看是否已经存在，如果存在则会产生冲突，那么就需要调用equals方法来比较两个对象是否相同，如果相同则覆盖掉原来的元素，如果不同，则需要处理冲突，具体过程可以参考源码，java是用链表的方法解决的。

下面是是hashmap put方法的源码：

```java
/*
Implements Map.put and related methods.
Params:
hash – hash for key
key – the key
value – the value to put
onlyIfAbsent – if true, don't change existing value
evict – if false, the table is in creation mode.
Returns:
previous value, or null if none
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    //((n - 1) & hash)确定了要put的元素的位置, 如果要插入的地方为空，就直接插入.
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //首先判断hash值是否相等，如果不相等，直接跳过，如果相等，会调用equals方法
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //将新加入的元素插入链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);//判断链表的长度，如果达到一定的值，则需要转化成红黑树
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

参考文章：https://www.cnblogs.com/tanshaoshenghao/p/10915055.html

https://www.cnblogs.com/skywang12345/p/3324958.html
