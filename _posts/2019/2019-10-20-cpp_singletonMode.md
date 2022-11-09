---
layout: single
related: false
title:  C++ 单例模式
date:   2019-10-20 23:39:00
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，主要是设计模式中的单例模式

# 1. 懒汉式单例模式

> 缺点是延迟加载，比如配置文件，只有在使用的时候才会加载。

```cpp
class CSingleton {
public:
    static CSingleton GetInstance() {
        if (m_pInstance == NULL)
            m_pInstance = new CSingleton();
        reutrn m_pInstance;
    }

private:
    CSingleton() {};

    static CSingleton * m_pInstace;
}
```

## 1.1. 多线程下的懒汉模式

> 使用double-check来保证线程安全。但是如果处理大量数据时，该锁才成为严重的性能瓶颈。

```cpp
class Singleton  
{  
private:  
    static Singleton* m_instance;  
    Singleton(){}  
public:  
    static Singleton* getInstance();  
};  
  
Singleton* Singleton::getInstance()  
{  
    if(NULL == m_instance)  
    {  
        Lock(); //借用其它类来实现，如boost  
        if(NULL == m_instance)  
        {  
            m_instance = new Singleton;  
        }  
        UnLock();  
    }  
    return m_instance;  
}
```

## 1.2. 饿汉式单例模式

> 一开始就创建实例对象并且加载，每次使用的时候直接返回就好了。  
> 饿汉式会出现线程安全问题，在多线程下，或个线程都初始化一个单例，得到的指针并不是指向同一个地方，就不满足单例类的定义，此时就需要进行修改。

```cpp
class CSingletonB {
private:
    CSingletonB () {}

public:
    static CSingletonB * GetInstance() {
        static CSingleton instance;
        return &instance;
    }
}
```
