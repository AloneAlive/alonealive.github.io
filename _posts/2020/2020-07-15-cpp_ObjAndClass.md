---
layout: single
related: false
title:  C++ 对象和类（案例代码）
date:   2020-07-15 23:52:00
categories: cpp
tags: cpp
toc: true
---

> C++类的声明、实现和使用，以及构造函数和析构函数。包含案例代码，可编译运行。

<!--more-->

# 1. 类的声明、实现、使用

## 1.1. 类的成员访问控制：公有/私有

无论类成员是数据成员还是成员函数，都可以在类的公有部分或私有部分中声明他。

而隐藏数据是OOP（面向对象）主要的目标之一，因此数据项通常放在私有部分，组成类接口的成员函数放在公有部分（否则就无法从程序中调用这些函数）

也可以把成员函数放在私有部分中，不能直接从程序中调用这种函数，但是公有方法却可以使用他们。通常，程序员使用私有成员函数处理不属于公有接口的实现细节。

类声明中的关键字private是类对象的默认访问控制，可以不用写出来。

```cpp
class A {
    float mass; //private by default
    ...
public:
    void call(void);
    ...
}
```

**Note：结构的默认访问类型是public，而类为private，可以不用再写出来。**

## 1.2. 类成员函数实现

类声明后，需要具体实现原型表示的成员函数。成员函数有函数头和函数体，也可以有返回类型和参数，而其和常规函数的不同之处：

+ 定义成员函数时，使用作用域解析运算符`::`来标识函数所属的类。例如`void A::update(double price)...`，这意味着`update()`函数是A类的成员。同时意味着可以将另一个类的成员函数也命名成`update()`，例如`void B::update()...`。因此作用域解析运算符确定了方法定义对应的类。
+ 类方法可以访问类的private私有成员，即定义的私有变量这些。

### 1.2.1. 案例

1. 头文件：

```cpp
//stock00.h
//stock00.h -- Stock class interfate
#ifndef STOCK00_H_
#define STOCK00_H_

#include <string>

class Stock {
private:   //can remove
	std::string company;
	long shares;
	double share_val;
	double total_val;
	//定义于类声明中的函数将自动成为内联函数
	//等价于在头文件的类外面使用inline void Stock::set_tot() {...}
	void set_tot() { total_val = shares * share_val; }

public:
        //对某公司股票的首次购买
	void acquire(const std::string & co, long n, double pr);
	//管理增加/减少持有的股票，确保买入或者卖出的股票不为负数
	void buy(long num, double price);
	void sell(long num, double price);
	void update(double price);
	void show();
};

#endif
```

2. 函数实现文件：

```cpp
//stock00.cpp
//stock00.cpp -- implementing the Stokc class
#include <iostream>
#include "stock00.h"

void Stock::acquire(const std::string & co, long n, double pr) {
	company = co;
	if (n < 0) {
		std::cout << "Number of shares can't be negative, "
			  << company << " shares set to 0.\n";
		shares = 0;
	} else
		shares = n;

	share_val = pr;
	set_tot();
}

void Stock::buy(long num, double price) {
	if (num < 0) {
		std::cout << "Number of shares purchased can't be negative, "
			  << "Transaction is aborted.\n";
	} else {
		shares += num;
		share_val = price;
		set_tot();
	}
}

void Stock::sell(long num, double price) {
	using std::cout;
	if (num < 0) {
		std::cout << "Number of shares purchased can't be negative, "
                          << "Transaction is aborted.\n";
	} else if (num > shares) {
	        std::cout << "You can't sell more than you have, "
                          << "Transaction is aborted.\n";
	} else {
		shares -= num;
		share_val = price;
		set_tot();
	}
}

void Stock::update(double price) {
	share_val = price;
	set_tot();
}

void Stock::show() {
	std::cout << "Company: " << company
		  << " Share: " << shares << '\n'
                  << " Share Price: $" << share_val
		  << " Total Worth: $" << total_val << '\n';
}
```

3. 类使用

```cpp
//usestock00.cpp 
//usestock00.cpp -- the client program
#include <iostream>
#include "stock00.h"

int main() {
	Stock f;
	f.acquire("NanoSmart", 20, 12.50);
	f.show();

	f.buy(15, 18.125);
	f.show();

	f.sell(400, 20.00);
	f.show();

	f.buy(300000, 40.125);
	f.show();

	f.sell(30000, 0.125);
	f.show();

	return 0;
}

```

4. 多个文件编译命令：

```
ubuntu@ubuntu-xm:~/Documents/sunwg/cpp$ g++ -c stock00.cpp
ubuntu@ubuntu-xm:~/Documents/sunwg/cpp$ g++ -c usestock00.cpp
ubuntu@ubuntu-xm:~/Documents/sunwg/cpp$ g++ stock00.o usestock00.o -o usestock00
```

5. 执行结果：

```s
ubuntu@ubuntu-xm:~/Documents/sunwg/cpp$ ./usestock00 
Company: NanoSmart Share: 20
 Share Price: $12.5 Total Worth: $250
Company: NanoSmart Share: 35
 Share Price: $18.125 Total Worth: $634.375
You can't sell more than you have, Transaction is aborted.
Company: NanoSmart Share: 35
 Share Price: $18.125 Total Worth: $634.375
Company: NanoSmart Share: 300035
 Share Price: $40.125 Total Worth: $1.20389e+07
Company: NanoSmart Share: 270035
 Share Price: $0.125 Total Worth: $33754.4

```

***

# 2. 类的构造函数和析构函数

## 2.1. 声明构造函数

构造函数专门用于构造新对象、将值赋给他们的数据成员。

例如:`Stock::Stock(const string & co, long n, double pr) {...}`

此处构造函数的参数表示的不是类成员，而是赋给类成员的值。因此，参数名不能与类成员相同，否则会造成混乱。

常见的做法是在数据成员名中使用`m_`前缀或者使用后缀`_`。

例如：

```cpp
class Stock {
	private:
		string m_company;
		long m_shares;
		...
}
```

或者

```cpp
class Stock {
	private:
		string company_;
		long shares_;
		...
}
```

## 2.2. 使用构造函数

两种使用构造函数初始化对象的方式：

1. 显式调用构造函数：`Stock food = Stock("World", 250, 1.25);`
2. 隐式调用构造函数：`Stock garment("Good", 50, 2.5);`，等价于显式的方法：`Stock garment = Stock("Good", 50, 2.5);`

创建类对象的时候，C++都会使用类的构造函数：`Stock *pstock = new Stock("Best", 18, 1.9);`

此处创建了一个Stock对象，并调用构造函数初始化为参数提供的值，将对象的地址赋给pstock指针。此时对象没有名称，但是可以使用指针来管理该对象。

使用对象调用方法：`stock1.show();`

## 2.3. 析构函数

如果构造函数使用new来分配内存，则析构函数将使用delete来释放这些内存。

如果Stock的构造函数没有使用new，则析构函数实际上没有需要完成的任务。在这种情况下，只需让编译器生成一个什么都不做的隐式析构函数即可。

析构函数可以没有返回值和声明类型。

和构造函数不同，析构函数没有参数。

例如Stock析构函数的原型：`~Stock();`

或者：

```cpp
Stock::~Sotck {
	cout << "Bye" <<endl;
}
```
