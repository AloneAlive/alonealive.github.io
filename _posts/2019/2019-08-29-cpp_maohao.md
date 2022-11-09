---
layout: single
related: false
title:  C++双冒号、点号、箭头的区别
date:   2019-08-29 21:52:00
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，包含C++双冒号、点号、箭头的区别

# 1. 箭头`->`和点号`.`

声明一个结构：

```cpp
struct mystruct {
    int age;
};
```

如果有个结构变量a，访问成员元素的方法：  
`a.age = 1;`

如果采用指针方法访问，则必须用箭头访问元素，比如:

```cpp
mystruct *ps;
ps->age = 1;
```

## 1.1. 指针对象

当定义类对象是指针对象的时候，需要用到`->`指向类中的成员；当定义一般对象的时候，使用`:`单冒号指向类中的成员。

```cpp
class A {
    public:
        play();
}

A *p;
p->play();  //左边是结构指针

A pr;
pr.play();  //左边是结构变量
```

# 2. 双冒号`::`

双冒号只用在类成员函数和类成员变量中。比如：

```cpp
class CA {
    public:
        int ca_var;
        int add(int a, int b);
}

//函数实现
int CA::add(int a, int b) {
    int c = ::ca_var;  //访问当前类实例中的变量
    return a+b;
}
```