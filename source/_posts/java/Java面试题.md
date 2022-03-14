---
title: Java面经（持续更新）
date: 2021-12-03
categories:
- Java
tags:
- Java知识点
language: zh-CN
toc: true
---

#### 抽象类和接口类的区别

- 抽象类要被子类继承，接口要被类实现；
- 抽象类只用做方法声明和实现，而接口只能做方法声明；
- 抽象类中的变量可以是普通变量，而接口中定义的变量必须是公共的静态常量；
- 抽象类是重构的结果，接口是设计的结果；
- 抽象类和接口都是用来抽象具体对象的，但是接口的抽象级别更高；
- 抽象类可以由具体的方法和属性，接口只能由抽象方法和静态常量。

<!--more-->

#### 为什么`HashMap`线程不安全

https://www.zhihu.com/question/28516433

####  值传递和引用传递(java)以及深拷贝和浅拷贝

#### hashmap的数组长度为什么是2的幂次方

https://blog.csdn.net/wohaqiyi/article/details/81161735