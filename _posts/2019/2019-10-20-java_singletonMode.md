---
layout: single
related: false
title:  Java 单例模式
date:   2019-10-20 22:59:00
categories: java
tags: java
toc: true
---

> 单例：保证一个类仅有一个实例，并提供一个访问它的全局访问点。  
> 单例模式是一种常用的软件设计模式之一，其目的是保证整个应用中只存在类的唯一个实例。  
> 比如我们在系统启动时，需要加载一些公共的配置信息，对整个应用程序的整个生命周期中都可见且唯一，这时需要设计成单例模式。如：spring容器，session工厂，缓存，数据库连接池等等。

**保证实例的唯一:**

1. 防止外部初始化
2. 由类本身进行实例化
3. 保证实例化一次
4. 对外提供获取实例的方法
5. 线程安全

# 1. 饿汉式单例模式

> 线程安全，调用效率高，但是不能延时加载

直接创建单例对象，使用的时候直接返回即可。缺点是单例在未使用的时候就已经初始化完成，如果程序一直没有使用，单例对象还是会创建，从而造成不必要的资源浪费。

```java
public class TestA {
    private static TestA t1 = new TestA;
    
    private TestA() {}

    public static TestA getInfo() {
        return t1;
    }
}
```

# 2. 懒汉式单例模式

> 线程安全，调用效率不高，但是可以延时加载

```java
public class TestB {
    //类初始化时，不初始化对象（延时加载，真正使用的时候再创建）
    private static TestB t2;

    //构造器私有化
    private TestB() {}

    //方法同步，调用效率低
    public static synchronized TestB getInfo() {
        if (testB == null) {
            TestB = new TestB();
        }
        return testB;
    }
}
```

***

# 3. 双重锁判断机制（DCL）

即Double CheckLock实现单例模式（由于JVM底层模型原因，偶尔会出现问题，不建议使用）

```java
public class TestC {
    private static TestC t3;
    
    private TestC() {}

    public static TestC getInfo() {
        if (TestC == null) {
            synchronized (TestC.class) {
                if (TestC == null) {
                    TestC = new TestC();
                }
            }
        }

        return t3;
    }
}
```

***

# 4. 静态内部类实现单例模式

> 线程安全，调用效率高，可以延迟加载

```java
public class TestD {

    private static class GetInfoClass {
        private static final TestD t4 = new TestD();
    }

    //构造函数
    private TestD() {}

    public static TestA getInfo() {
        return GetInfoClass.t4;
    }
}
```

***

# 5. 枚举类实现单例模式

> 线程安全，调用效率高，不能延时加载，可以天然防止反射和反序列化调用）

```java
public class TestE {
  
    //枚举元素本身就是单例
    INSTANCE;

    //添加自己需要的操作
    public void getInfoOperation() {
    }
}
```
