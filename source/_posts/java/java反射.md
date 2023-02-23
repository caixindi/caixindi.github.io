---
title: java反射
date: 2023-02-23
categories:
- Java
tags:
- Java知识点
language: zh-CN
toc: true
---
### 反射

> Java反射机制是在运行状态中，对于任何一个类，都能获得它的所有属性和方法，对于任何一个对象，都能够调用它的任意一个方法和属性。

RTTI（Run-Time Type Identification）运行时类型识别。其作用是在运行时识别一个对象的类型和类的信息。主要有两种方式：一种是“传统的”RTTI，它假定我们在编译时已经知道了所有的类型；另一种是“反射”机制，它允许我们在运行时发现和使用类的信息。反射就是把java类中的各种成分映射成一个个的Java对象。如果要动态获取到这些信息，需要依靠`Class`对象。`Class`类对象将一个类的方法、变量等信息告诉运行的程序。
<!--more-->

#### 如何获取Class对象

1. 通过类名

   ```java
   Class clazz = TargetObject.class;
   ```

2. 通过类的全限定名

   ```java
   Class clazz = Class.forName("com.example.TargetObject");
   ```

3. 通过对象实例

   ```java
   Class clazz = instance.getClass();
   ```

4. 通过类加载器

   ```java
   Class clazz = ClassLoader.getSystemClassLoader().loadClass("com.example.TargetObject");
   ```

   > 通过类加载器获取Class对象不会进行初始化，意味着不进行包括初始化等一系列步骤，静态代码块和静态对象不会得到执行

**Class类的方法**

| 方法名                    | 说明                                                         |
| :------------------------ | :----------------------------------------------------------- |
| forName()                 | 返回一个给定类或者接口的一个`Class`对象，如果没有给定classloader，那么会使用根类加载器。如果 initalize 参数为true（默认为true），那么给定的类如果之前没有被初始化过，那么会被初始化。 |
| getClass()                | 获取Class对象的一个引用，返回表示该对象的实际类型的Class引用。 |
| getName()                 | 获取类的全限定名。                                           |
| getSimpleName()           | 获取类名（不包括包名）。                                     |
| getCanonicalName()        | 获取全限定的类名(包括包名)。                                 |
| isInterface()             | 判断该class对象是否为接口。                                  |
| getInterfaces()           | 返回Class对象数组，表示Class对象所引用的类所实现的所有接口。 |
| getSuperclass()           | 返回Class对象，表示Class对象所引用的类所继承的直接基类。     |
| newInstance()             | 返回Object对象，用于创建类的新实例，要求该类拥有无参构造函数。 |
| getFields()               | 获得该类所有的公共（public）的字段，包括继承自父类的所有公共字段。 |
| getMethods()              | 返回一个包含Method对象的数组，该对象反映了该Class对象所表示的类或接口的所有公共成员方法，包括类或接口声明的方法以及从超类和超接口继承的方法。 |
| getConstructors()         | 返回一个包含Constructor对象的数组，即获取该类的所有公共构造函数。 |
| getDeclaredFields()       | 获得某个类声明的字段，即包括public、private和proteced，但是不包括父类声明的任何字段。 |
| getDeclaredMethods()      | 返回一个包含Method对象的数组，该对象反映了该Class对象所表示的类或接口的所有成员方法，包括类或接口声明的方法。 |
| getDeclaredConstructors() | 返回一个包含Constructor对象的数组，即获取该类的所有构造函数。 |

> getName、getSimpleName、getCanonicalName的区别
>
> getName只获取类名，getSimpleName和getCanonicalName是获取类的全限定名，大部分情况下其返回结果相同，但是在数组、内部类的情况下有所不同。

#### Constructor类

>Constructor类存在于反射包（java.lang.reflect）中，反映的是Class对象所表示的类构造方法。

获取Constructor对象是通过Class类中的方法实现的，Class类与Constructo类相关的主要方法如下：

| 方法名                                             | 说明                                                      |
| -------------------------------------------------- | --------------------------------------------------------- |
| forName(String className)                          | 返回带有给定字符串名的类或接口相关联的Class对象           |
| getConstructor(Class<?>... parameterTypes)         | 返回指定参数类型、具有public访问权限的构造函数对象        |
| Constructor<?>[]                                   | 返回所有具有public访问权限的构造函数的Constructor对象数组 |
| getDeclaredConstructor(Class<?>... parameterTypes) | 返回指定参数类型、所有声明的（包括private）构造函数对象   |
| getDeclaredConstructors()                          | 返回所有声明的（包括private）构造函数对象                 |
| newInstance()                                      | 调用无参构造器创建此 Class 对象所表示的类的一个新实例。   |

#### Field类及其用法

>Field提供有关类或接口的单个字段信息，并且可以获得动态访问权限。反射的字段可能是一个类（静态）或实例字段。

获取类的字段Field是通过Class类中的方法实现的，Class类与Field类相关的主要方法如下：

| 方法名                        | 说明                                                      |
| ----------------------------- | --------------------------------------------------------- |
| getDeclaredField(String name) | 获取指定名称的字段，不包括继承的字段                      |
| getDeclaredFields()           | 获取Class对象所表示的类或接口的所有字段，不包括继承字段   |
| getField(String name)         | 获取指定名称的公共字段，包括继承字段                      |
| getFields()                   | 获取Class对象所表示的类或接口的所有公共字段，包括继承字段 |

> 需要注意的是，如果我们不希望获取父类的字段，则需要使用Class类的getDeclaredField或者getDeclaredFields方法来获取字段，如果需要获取父类的字段，则需要使用getField或者getFields，但也只能获取到公共字段，无法获取私有字段。对于私有字段，需要使用setAccessible方法将该字段设置为可访问。

Field类的常用方法如下：

| 方法名                        | 说明                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| set(Object obj, Object value) | 将指定对象变量上此 Field 对象表示的字段设置为指定的新值。    |
| get(Object obj)               | 返回指定对象上此 Field 表示的字段的值                        |
| getType()                     | 返回一个 Class 对象，它标识了此Field 对象所表示字段的声明类型。 |
| isEnumConstant()              | 如果此字段表示枚举类型的元素则返回 true；否则返回 false      |
| toGenericString()             | 返回一个描述此 Field（包括其一般类型）的字符串               |
| getName()                     | 返回此 Field 对象表示的字段的名称                            |
| getDeclaringClass()           | 返回表示类或接口的 Class 对象，该类或接口声明由此 Field 对象表示的字段 |
| setAccessible(boolean flag)   | 将此对象的 accessible 标志设置为指示的布尔值,即设置其可访问性 |

> 被final关键字修饰的Field字段是安全的，在运行时虽然可以接收任何修改，但最终其实际值是不会发生改变的。

#### Method类及其用法

> Method类提供关于类或接口上单独某个方法（以及如何访问该方法）的信息，所反映的方法可能是类方法或实例方法（包括抽象方法）。

获取类的方法Method是通过Class类中的方法实现的，Class类与Method类相关的主要方法如下：

| 方法名                                                     | 说明                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| getDeclaredMethod(String name, Class<?>... parameterTypes) | 返回一个指定参数的Method对象，该对象反映此Class对象所表示的类或接口的指定已声明方法。 |
| getDeclaredMethods()                                       | 返回Method对象的一个数组，这些对象反映此Class对象表示的类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。 |
| getMethod(String name, Class<?>... parameterTypes)         | 返回一个指定参数的Method对象，该对象反映此Class对象所表示的类或接口的指定已声明的公共成员方法，包括继承的方法。 |
| getMethods()                                               | 返回Method对象的一个数组，这些对象反映此 Class 对象所表示的类或接口（包括那些由该类或接口声明的以及从超类和超接口继承的那些的类或接口）的公共成员方法。 |

Method类常用的方法如下：

| 方法名                             | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| invoke(Object obj, Object... args) | 对带有指定参数的指定对象调用由此Method对象表示的方法。       |
| getReturnType()                    | 返回一个Class对象，该对象描述了此Method对象所表示的方法的返回类型,即方法的返回类型 |
| getGenericReturnType()             | 返回表示由此Method对象所表示方法的正式返回类型的 Type 对象，也是方法的返回类型。 |
| getParameterTypes()                | 按照声明顺序返回Class对象的数组，这些对象描述了此Method对象所表示的方法的形参类型。即返回方法的参数类型组成的数组 |
| getGenericParameterTypes()         | 按照声明顺序返回Type对象的数组，这些对象描述了此Method对象所表示的方法的形参类型的，也是返回方法的参数类型 |
| getName()                          | 以String形式返回此Method对象表示的方法名称，即返回方法的名称 |
| isVarArgs()                        | 判断方法是否带可变参数，如果将此方法声明为带有可变数量的参数，则返回true；否则，返回false。 |
| toGenericString()                  | 返回描述此 Method 的字符串，包括类型参数。                   |

### 反射机制的执行流程

#### 反射获取类实例

首先调用`java.lang.Class`的forName方法，获取类信息

> forName()首选通过反射获取类信息，通过jvm去完成
>
> 首先获取调用类的ClassLoader, 然后调用native方法，获取给定name的类信息，最后回调java.lang.ClassLoader.loadClass进行类加载

```java
@CallerSensitive
public static Class<?> forName(String className)
            throws ClassNotFoundException {
    // 先通过反射区获取调用进来的类信息 getCallerClass为native方法
    Class<?> caller = Reflection.getCallerClass();
    // 调用native方法进行获取class信息
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}
```

接下来是`newInstance()`方法：

>`newInstance()`主要工作如下：
>
>- 权限检测，如果不通过直接抛出异常；
>- 查找无参构造器，并将其缓存起来；
>- 调用具体方法的无参构造方法，生成实例并返回；

```java
@CallerSensitive
public T newInstance()
    throws InstantiationException, IllegalAccessException
{
    if (System.getSecurityManager() != null) {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
    }

    // NOTE: the following code may not be strictly correct under
    // the current Java memory model.

    // Constructor lookup
    // 寻找构造函数（无参构造函数） 
    // 如果没有缓存
    if (cachedConstructor == null) {
        if (this == Class.class) {
            // 不能调用Class类的newInstance()方法
            throw new IllegalAccessException(
                "Can not call newInstance() on the Class for java.lang.Class"
            );
        }
        try {
            // 寻找无参构造器
            Class<?>[] empty = {};
            final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
            // Disable accessibility checks on the constructor
            // since we have to do the security check here anyway
            // (the stack depth is wrong for the Constructor's
            // security check to work)
            // 禁用构造函数的可访问性检查，因为无论如何都必须在这里进行安全检查
            java.security.AccessController.doPrivileged(
                new java.security.PrivilegedAction<Void>() {
                    public Void run() {
                            c.setAccessible(true);
                            return null;
                        }
                    });
            // 存入缓存
            cachedConstructor = c;
        } catch (NoSuchMethodException e) {
            throw (InstantiationException)
                new InstantiationException(getName()).initCause(e);
        }
    }
    Constructor<T> tmpConstructor = cachedConstructor;
    // Security check (same as in java.lang.reflect.Constructor)
    int modifiers = tmpConstructor.getModifiers();
    if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
        Class<?> caller = Reflection.getCallerClass();
        if (newInstanceCallerCache != caller) {
            Reflection.ensureMemberAccess(caller, this, null, modifiers);
            newInstanceCallerCache = caller;
        }
    }
    // Run constructor
    try {
        return tmpConstructor.newInstance((Object[])null);
    } catch (InvocationTargetException e) {
        Unsafe.getUnsafe().throwException(e.getTargetException());
        // Not reached
        return null;
    }
}
```

下面是获取类构造器的过程：

> getConstructor0() 为获取匹配的构造方器；分三步：
>
> - 先获取所有的constructors, 然后通过进行参数类型比较；
> - 找到匹配后，通过 ReflectionFactory 拷贝一份constructor返回；
> - 否则抛出 NoSuchMethodException;

```java
private Constructor<T> getConstructor0(Class<?>[] parameterTypes,
                                    int which) throws NoSuchMethodException
{
    Constructor<T>[] constructors = privateGetDeclaredConstructors((which == Member.PUBLIC));
    for (Constructor<T> constructor : constructors) {
        if (arrayContentsEq(parameterTypes,
                            constructor.getParameterTypes())) {
            return getReflectionFactory().copyConstructor(constructor);
        }
    }
    throw new NoSuchMethodException(getName() + ".<init>" + argumentTypesToString(parameterTypes));
}
```

>privateGetDeclaredConstructors(), 获取所有的构造器的大致步骤为：
>
>- 先尝试从缓存中获取；
>- 如果缓存没有，则通过jvm获取，获取之后存入缓存，缓存使用软引用进行保存，保证内存可用

```java
// 获取当前类所有的构造方法，通过jvm或者缓存
// Returns an array of "root" constructors. These Constructor
// objects must NOT be propagated to the outside world, but must
// instead be copied via ReflectionFactory.copyConstructor.
private Constructor<T>[] privateGetDeclaredConstructors(boolean publicOnly) {
    checkInitted();
    Constructor<T>[] res;
    // 调用 reflectionData(), 获取保存的信息，使用软引用保存，从而使内存不够可以回收
    ReflectionData<T> rd = reflectionData();
    if (rd != null) {
        res = publicOnly ? rd.publicConstructors : rd.declaredConstructors;
        // 存在缓存，则直接返回
        if (res != null) return res;
    }
    // No cached value available; request value from VM
    if (isInterface()) {
        @SuppressWarnings("unchecked")
        Constructor<T>[] temporaryRes = (Constructor<T>[]) new Constructor<?>[0];
        res = temporaryRes;
    } else {
        // 使用native方法从jvm获取构造器
        res = getDeclaredConstructors0(publicOnly);
    }
    if (rd != null) {
        // 最后，将从jvm中读取的内容，存入缓存
        if (publicOnly) {
            rd.publicConstructors = res;
        } else {
            rd.declaredConstructors = res;
        }
    }
    return res;
}

// Lazily create and cache ReflectionData
private ReflectionData<T> reflectionData() {
    SoftReference<ReflectionData<T>> reflectionData = this.reflectionData;
    int classRedefinedCount = this.classRedefinedCount;
    ReflectionData<T> rd;
    if (useCaches &&
        reflectionData != null &&
        (rd = reflectionData.get()) != null &&
        rd.redefinedCount == classRedefinedCount) {
        return rd;
    }
    // else no SoftReference or cleared SoftReference or stale ReflectionData
    // -> create and replace new instance
    return newReflectionData(reflectionData, classRedefinedCount);
}

// 新创建缓存，保存反射信息 使用SoftReference 软引用
private ReflectionData<T> newReflectionData(SoftReference<ReflectionData<T>> oldReflectionData,
                                            int classRedefinedCount) {
    if (!useCaches) return null;
    // 使用cas保证更新的线程安全性，所以反射是保证线程安全的
    while (true) {
        ReflectionData<T> rd = new ReflectionData<>(classRedefinedCount);
        // try to CAS it...
        if (Atomic.casReflectionData(this, oldReflectionData, new SoftReference<>(rd))) {
            return rd;
        }
        // 先使用CAS更新，如果更新成功，则立即返回，否则测查当前已被其他线程更新的情况，如果和自己想要更新的状态一致，则也算是成功了
        oldReflectionData = this.reflectionData;
        classRedefinedCount = this.classRedefinedCount;
        if (oldReflectionData != null &&
            (rd = oldReflectionData.get()) != null &&
            rd.redefinedCount == classRedefinedCount) {
            return rd;
        }
    }
}
```

如何得到所要获取的构造器：

> 比较类型 只要有其中一个类型不相等 则直接返回false

```java
private static boolean arrayContentsEq(Object[] a1, Object[] a2) {
    if (a1 == null) {
        return a2 == null || a2.length == 0;
    }

    if (a2 == null) {
        return a1.length == 0;
    }

    if (a1.length != a2.length) {
        return false;
    }

    for (int i = 0; i < a1.length; i++) {
        if (a1[i] != a2[i]) {
            return false;
        }
    }

    return true;
}
// sun.reflect.ReflectionFactory
/** Makes a copy of the passed constructor. The returned
    constructor is a "child" of the passed one; see the comments
    in Constructor.java for details. */
public <T> Constructor<T> copyConstructor(Constructor<T> arg) {
    return langReflectAccess().copyConstructor(arg);
}

// java.lang.reflect.Constructor, copy 其实就是新new一个 Constructor 出来
Constructor<T> copy() {
    // This routine enables sharing of ConstructorAccessor objects
    // among Constructor objects which refer to the same underlying
    // method in the VM. (All of this contortion is only necessary
    // because of the "accessibility" bit in AccessibleObject,
    // which implicitly requires that new java.lang.reflect
    // objects be fabricated for each reflective call on Class
    // objects.)
    if (this.root != null)
        throw new IllegalArgumentException("Can not copy a non-root Constructor");

    Constructor<T> res = new Constructor<>(clazz,
                                           parameterTypes,
                                           exceptionTypes, modifiers, slot,
                                           signature,
                                           annotations,
                                           parameterAnnotations);
    // root 指向当前 constructor
    res.root = this;
    // Might as well eagerly propagate this if already present
    res.constructorAccessor = constructorAccessor;
    return res;
}
```

得到构造器之后，需要调用构造器的`newInstance()`方法，即可得到实例

> 得到实例之后需要进行类型转换

```java
@CallerSensitive
public T newInstance(Object ... initargs)
    throws InstantiationException, IllegalAccessException,
           IllegalArgumentException, InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, null, modifiers);
        }
    }
    if ((clazz.getModifiers() & Modifier.ENUM) != 0)
        throw new IllegalArgumentException("Cannot reflectively create enum objects");
    ConstructorAccessor ca = constructorAccessor;   // read volatile
    if (ca == null) {
        ca = acquireConstructorAccessor();
    }
    @SuppressWarnings("unchecked")
    T inst = (T) ca.newInstance(initargs);
    return inst;
}
```

#### 反射获取方法

首先获取Method:

1. 权限检查
2. 获取所有方法列表
3. 根据方法名以及方法参数获取符合条件的方法
4. 如果没有找到相关方法，则抛出异常，否则将找到的方法返回

```java
@CallerSensitive
public Method getDeclaredMethod(String name, Class<?>... parameterTypes)
    throws NoSuchMethodException, SecurityException {
    checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
    Method method = searchMethods(privateGetDeclaredMethods(false), name, parameterTypes);
    if (method == null) {
        throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
    }
    return method;
}
```

获取类所有声明的方法需要使用`privateGetDeclaredMethods(fasle)`方法：

>大致过程与获取构造器方法类似：
>
>- 先尝试从缓存中获取；
>- 如果缓存没有，则通过jvm获取，获取之后存入缓存，缓存使用软引用进行保存，保证内存可用

```java
// Returns an array of "root" methods. These Method objects must NOT
// be propagated to the outside world, but must instead be copied
// via ReflectionFactory.copyMethod.
private Method[] privateGetDeclaredMethods(boolean publicOnly) {
    checkInitted();
    Method[] res;
    ReflectionData<T> rd = reflectionData();
    if (rd != null) {
        res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
        if (res != null) return res;
    }
    // No cached value available; request value from VM
    res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
    if (rd != null) {
        if (publicOnly) {
            rd.declaredPublicMethods = res;
        } else {
            rd.declaredMethods = res;
        }
    }
    return res;
}
```

然后根据方法名和参数类型返回指定方法：

> 方法名以及参数类型符合则返回该方法，但是返回的是最精确匹配的方法返回，最后通过`copyMethod()`方法返回

```java
private static Method searchMethods(Method[] methods,
                                    String name,
                                    Class<?>[] parameterTypes)
{
    Method res = null;
    // 使用常量池，避免重复创建String
    String internedName = name.intern();
    for (int i = 0; i < methods.length; i++) {
        Method m = methods[i];
        if (m.getName() == internedName
            && arrayContentsEq(parameterTypes, m.getParameterTypes())
            && (res == null
                // 判断当前循环方法返回类型是否是当前匹配到方法返回类型的子类 或者类型相同
                // 满足条件 替换res 使得匹配更精确
                || res.getReturnType().isAssignableFrom(m.getReturnType())))
            res = m;
    }

    return (res == null ? res : getReflectionFactory().copyMethod(res));
}
```

#### 调用`method.invoke()`方法

> invoke 是通过 MethodAccessor 进行调用的，而 MethodAccessor 是个接口，在第一次时调用 acquireMethodAccessor() 进行创建。

```java
@CallerSensitive
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
```

执行 `ma.invoke(obj, args)`时，调用 `DelegatingMethodAccessorImpl.invoke()`最后被委托到 `NativeMethodAccessorImpl.invoke()`, 即：

```java
public Object invoke(Object obj, Object[] args) throws IllegalArgumentException, InvocationTargetException{
    // We can't inflate methods belonging to vm-anonymous classes because
    // that kind of class can't be referred to by name, hence can't be
    // found from the generated bytecode.
    if (++numInvocations > ReflectionFactory.inflationThreshold()
            && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
        MethodAccessorImpl acc = (MethodAccessorImpl)
            new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        parent.setDelegate(acc);
    }

    // invoke0 是个 native 方法，由jvm进行调用业务方法。从而完成反射调用功能。
    return invoke0(method, obj, args);
}
```

#### 反射的应用场景

平时写业务代码的时候一般很少会用到需要反射机制的场景，但反射在框架中被大量使用，比如Spring Boot、MyBaits等等。例如Java中的动态代理实现也是依赖了反射的机制。

比如下面的实现动态代理的示例代码：

```java
public class DebugInvocationHandler implements InvocationHandler {

    // 代理类中的真实对象
    private final Object target;

    public DebugInvocationHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        System.out.println("before method " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("after method " + method.getName());
        return result;
    }
}
```

#### 反射调用流程总结

反射实现原理大致有如下几点：

1. 反射类及反射方法的获取，都是通过从列表中搜寻查找匹配的方法，所以查找性能会随类的大小方法多少而变化；
2. 每个类都会有一个与之对应的Class实例，从而每个类都可以获取method反射方法，并作用到其他实例身上；
3. 反射也是考虑了线程安全的；
4. 反射使用软引用relectionData缓存class信息，避免每次重新从jvm获取的重复开销；
5. 反射调用多次生成新代理Accessor, 而通过字节码生存的则考虑了卸载功能，所以会使用独立的类加载器；
6. 当找到需要的方法，都会copy一份出来，而不是使用原来的实例，从而保证数据隔离；
7. 调度反射方法，最终是由jvm执行invoke0()执行；
