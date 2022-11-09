---
layout: single
related: false
title:  C++内存分配方式和模板类vector, array
date:   2019-08-13 23:31:05
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，包含C++内存分配方式和模板类vector, array

# 1. 使用new创建动态结构

> 在运行时创建数组优于编译时创建数组，对于结构也是如此。  
> 需要在程序运行时为结构分配所需的空间，可以使用`new`完成。  
> 动态意味着内存是在`运行时`，而不是`编译时`分配的。  
> 例如`inflatable *ps = new inflatable; `其中inflatable是一个结构类型。这句代码将把存储结构inflatable的一块可用内存的地址赋值给ps。  
> **箭头成员运算符**`->`，可用于指向结构的指针。例如ps指向一个inflatable结构的成员price，即`ps->price`
<!--more-->
**Note：**  
+ 如果结构标识符是结构名，则使用句点运算符
+ 如果标识符是指向结构的指针，则使用箭头运算符

**示例代码：**

```cpp
// newstruct.cpp 
#include<iostream>

using namespace std;

struct inflatable {
	char name[20];
	float volume;
	double price;
};

int main() {
	inflatable *ps = new inflatable;
	cout << "Enter name: ";
	cin.get(ps->name, 20);

	cout << "Enter volume:";
	cin >> (*ps).volume;

	cout << "Enter price: $";
	cin >> ps->price;

	cout << "Result: Name=" << (*ps).name <<endl;
	cout << "Volume=" << ps->volume <<endl;
	cout << "Price=" << ps->price <<endl;

	delete ps;  //删除new创建的对象
	return 0;
}
```

**执行结果**：

```shell
Enter name: Peter
Enter volume:27.99
Enter price: $23.54
Result: Name=Peter
Volume=27.99
Price=23.54
```

**示例代码：**

```cpp
//delete.cpp 
#include<iostream>
#include<cstring>

using namespace std;

char * getname(void);  //一般可以放在头文件

int main()
{
	char * name;

	name = getname();
	cout << name << " at " << (int*)name <<endl;
	delete []name;

	name = getname();
	cout << name << " at " << (int*)name <<endl;
	delete []name;

	return 0;
}

char * getname() {
	char temp[80];  //暂时内存
	cout << "Enter name: ";
	cin >> temp;

	char * pn = new char[strlen(temp)+1];  //+1包含空字符
	strcpy(pn, temp);

	return pn;  //返回指针
}
```

**执行结果：**

```shell
Enter name: peter
peter at 0x21ee440
Enter name: nancy
nancy at 0x21ee440
```

# 2. 管理数据内存的四种方式

> 根据用于分配内存的方法，C++有三种管理数据内存的方式：`自动存储、静态存储、动态存储（有时候也叫做自由存储空间或堆）`  
> C++11增加了第四种类型：`线程存储`

## 2.1. 自动存储(stack)

在函数内部定义的常规变量使用自动存储空间，被称为自动变量。这意味着他们所属的函数被调用时自动产生，函数结束时消亡。  
实际上，自动变量是一个局部变量，其作用域是包含他的代码块。（代码块是被包含在花括号中的一段代码）  
`自动变量`通常存储在`栈`中，这意味着执行代码时，其中的变量将依次加入到栈中，而在离开代码块时，按照相反的顺序释放这些变量（`先进后出LIFO`）。因此，在程序执行过程中，栈将不断地的增大和缩小。

## 2.2. 静态存储

静态存储是整个程序执行期间都存在的存储方式。**使变量成为静态的方式有两种**：
+ 在函数外面定义它
+ 在声明变量时使用关键字`static`  (例如`static double fee = 56.30;`)

**自动存储和静态存储的关键在于**： 这些方法严格的限制了变量的寿命。变量可能存在于程序的整个生命周期（静态变量），也有可能只在特定函数被执行时存在（自动变量）

## 2.3. 动态存储(heap or free store)

new和delete提供了一种比前两者更加灵活的方法。他们管理了一个内存池，这个在C++中被称为**自由存储空间free store**或者**堆heap**  
该内存池同用于静态变量和自动变量的内存是分开的。  
new和delete能够让你在一个函数中分配内存，而在另一个函数中释放它。

## 2.4. 有关栈、堆和内存泄漏

> 如果使用new在堆（或者自由存储空间）上创建变量后，没有调用delete。将会发生什么情况呢？

如果没有调用delete，则即使包含指针的内存由于作用域规则和对象生命周期的原因被释放，在堆上动态分配的变量或者结构还是会继续存在。  
实际上，将会无法方位堆中的结构，因为指向这些内存的指针无效。这将会导致`内存泄漏memory leak`。  
被泄漏的内存在程序的整个生命周期都不能使用，这些内存被分配出去，但是无法收回。 极端情况下，内存泄漏可能会非常严重，以致于应用程序的可用内存被耗尽，导致`程序崩溃crash`。
**因此为比描内存泄漏，同时使用new和delete运算符**

# 3. 类型组合

```cpp
struct yearinfo
{
    int year;
};

yearinfo s1, s2, s3;  //都是结构
s0.year = 1996;  //使用句点运算符访问成员

yearinfo *pa = &s0;  //创建指向这个结构变量的指针（地址运算符）
pa->year = 1999;  //使用指针的箭头运算符访问成员

yearinfo info[3];  //创建结构数组
info[0].year = 2019;  
(info+1)->year = 2004; //地址加一（一个类型的字节数） ， 同info[1].year

const yearinfo * arp[3] = {&s0, &s1, &s2};
arp[1]->year   //访问成员

const yearinfo ** ppa = arp;  //建立上述指针的指针

auto ppb = arp;  //C++11版本的auto能够正确的推断ppb的类型

(*ppa)->year
(*(ppb+1))->year
```

## 3.1. 数组的替代品`vector`

模板类vector类似于string类，也是一种动态数组。
**基本使用：**
1. 必须包含头文件vector
2. 包含在命名空间std中
3. 模板使用不同的语法来指出它存储的数据类型
4. 使用不同的语法来指定元素数

**示例代码：**

```cpp
#include<vector>
...
using namespace std;
vector<int> vi;
int n;
cin >> n;
vector<double> vd(n);  //创建数组包含n个double元素
```

**缺点：**
效率相比数组稍低；而数组长度固定，不方便和安全。

# 4. C++11新增模板类`array`

位于命名空间std中，并且长度固定，也是使用栈（静态内存分配），因此效率和数组相同。  
创建需要包含头文件`array`。

```cpp
#include<array>
...
using namespace std;
array<int, 5> ai;  //创建长度是5，类型是int的array模板
array<double, 4> ad = {1.2, 32.2, 31.2, 3.2};
```

# 5. 复杂类型的小结

**结构**可以将多个不同类型的值存储在同一个数据对象中，可以使用成员关系运算符(.)访问成员。

**共同体**可以存储一个值，但是这个值可以是不同的类型，成员名指出了使用的模式。

**指针**是被设计用来存储地址的变量。指针声明指出了指针指向的对象的类型。指针指向了它存储的地址。对指针应用接触引用运算符，将得到指针指向的位置中的值。

**字符串**以空字符为结尾的一系列字符。字符串可用引号括起的字符串常量表示，其中隐式包含了结尾的空字符。可以将字符串存储在char数组中，可以用被初始化为指向字符串的char指针表示字符串。

函数strlen()返回字符串长度，但是不包含空字符。

函数strcpy()将字符串从一个位置复制到另一个位置。需要加入头文件cstring或者string.h

**new运算符**允许在程序运行时为数据对象请求内存。该运算符返回获得内存的地址，可以将这个地址赋值给一个指针，程序将只能使用该指针来访问这块内存。

+ 如果是简单变量，使用解除引用运算符`*`来获取值；
+ 如果数据对象是数组，则可以使用数组名那样使用指针来访问元素；
+ 如果数据对象是结构，使用`->`访问成员

**指针和数组紧密相关**，如果ar是数组名，则表达式`ar[i]`被解释为`*(ar+i)`，其中数组名被解释为数组第一个元素的地址。这样，数组名的作用同指针。反之，可以使用数组表示法，通过指针名来访问new分配的数组中的元素。
