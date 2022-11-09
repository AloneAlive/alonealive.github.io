---
layout: single
related: false
title: C++浮点常量表示、算术运算符、类型转换
date:   2019-07-25 22:01:37
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，主要讲述C++浮点常量表示、算术运算符、类型转换

# 1. 浮点常量

> 默认情况下，`8.27`和`3.4E5`这类浮点常量都属于double类型。  
> 如果希望常量是float类型，使用`f`或者`F`后缀  
> 对于`long double`的类型，使用`l`和`L`的后缀

__例如：__

```cpp
1.234f  //float
2.2L    //long double
2.34F   //float
23.231E24  //double (defalut)
```

# 2. 算术运算符

## 2.1. 五种C++基本运算符

operation|note|example
:-:|:-:|:-:
+|加法|`3+43=46`
-|减法|`20-5=15`
*|乘以|`3*5=15`
/|除以|`10/3=3`（整型相除，小数部分丢弃）
%|取余|`10%3=1`

__例如：__

```cpp
// arith.cpp
#include<iostream>

using namespace std;

int main() {
	float a = 50.25f;
	float b = 11.17f;

	cout<< "a=" << a << ",b="<<b<<endl;
	cout<< "a+b="<< a+b <<endl;
	cout<< "a-b="<< a-b <<endl;
	cout<< "a*b="<< a*b <<endl;
	cout<< "a/b="<< a/b <<endl;
	return 0;
}
```

__结果：__

```cpp
// Result
a=50.25,b=11.17
a+b=61.42
a-b=39.08
a*b=561.292
a/b=4.49866
```

## 2.2. 除法

> 如果两个整数相除，结果的小数部分将被丢弃，只保留整数部分  
>
> 如果其中一个（或者两个）是浮点数，则结果也会保留小数部分，类型是浮点数  

# 3. 类型转换

> <font color=red>C++自动执行的类型转换：</font>
> + 将一个算数类型的值赋给另一种算数类型的变量时，C++会对值转换  
> + 表达式包含不同的类型时，C++会对值转换  
> + 将参数传递给函数时，C++会对值转换 

## 3.1. 类型转换的潜在问题

转换|潜在问题
:-:|:-:
将较大的浮点类型转换成较小的浮点类型，例如double转换成float|精度（有效位数降低），值可能超出目标类型的取值范围
将浮点类型转换成整型|小数部分丢失
将较大的整型转换成较小的整型，例如long转换成short|值可能超出目标类型的取值范围，通常<b>只复制右边的字节</b>

> 将0赋值给bool类型，被转换成true  
> 非零的则是false

## 3.2. 以`{}`方式初始化时进行的转换（C++11）

> C++11将使用`{}`的初始化称为`列表初始化`  
> 因为这种初始化常用于给复杂的数据类型提供值列表  
> 列表初始化不允许`缩窄`，即可能无法赋值。例如浮点型不允许转换成整型

```diff
const int code = 66;  //整型常量
int x = 66;   //整型变量

- char c1 {31366};  //类型缩窄，非法
+ char c2 = {66}; //合法
+ char c3 {code};  //合法
- char c4 = {x};  //非法，x是一个变量，对于编译器而言，这个值可能很大。所以编译器不会跟踪执行以下阶段：从x被初始化，到被用来初始化c4

x = 31355;
+ char c5 = x;  //合法，因为x已经被初始化
```

***

## 3.3. 表达式中的`自动转换`

> C++在表达式会执行两种自动转换：
> 1. 类型出现时就自动转换
> 2. 类型和其他类型同时出现在表达式中，将被转换

+ __整型提升__  
在计算表达式时，C++将 __bool,char,unsigned char,signed char,short__ 转换成`int`  
其中`true -> 1, false -> 0` 

```cpp
short a =20;
short b =30;
short c = a+b; //在进行计算的时候，会先将a,b转换成int，然后将结果转换成short
```
__Note:__ 因为int是计算机最自然的类型，计算机使用这种类型时，运算速度可能最快。

> 如果short比int短，则unsigned short会被转换成int；  
> 如果两个长度相同，unsigned short会被转换成unsigned int；  
> 这种规则保证unsigned short进行整型提升的时候不会损失数据

***

+ __C++11校验后，编译器在进行算术运算时，依次如下查阅：__

> 如果有一个操作类型是`long double`, 则另一个转换成`long double`  
> 否则，有一个是`double`， 另一个转换成`double`  
> 否则，有一个是`float`， 另一个转换成`float`  
> 否则，说明操作数都是整型，执行整型提升  
> + 在这种情况下，如果两个操作数都是有符号或者无符号的，且级别不同，则转换成级别高的类型  
> + 如果一个操作数有符号，另一个没有符号，且无符号操作数的级别比有操作符的高，则将有符号操作数转换成无符号操作数（还是转向级别高的类型）
> + 否则，`有符号类型可以表示无符号类型的所有可能取值`，则将无符号操作数转换成有符号操作数的类型
> + __否则，将两个操作数都转换成有符号类型的无符号版本__

***

## 3.4. 传递参数的时候转换类型

> 传递参数时的类型转换 通常由`C++函数原型控制`  
> C++将对char和short类型应用整型提升（int）  

## 3.5. __强制类型转换__

> 强制转换不会修改原变量本身，而是创建一个新的、指定类型的值

+ 通用格式如下：

```cpp
(typename) value    //C的方式
typename (value)    //C++的方式

例如将变量a的int值转换成long类型：
(long) a
long (a)
```

+ 强制类型转换符

```cpp
static_cast<typename> (value)

例如：
static_cast<long> (a)
```

***

# 4. __C++11的`auto`声明__

> auto是C++的关键字  
> 使用auto，而不指定变量类型，编译器将把变量的类型设置成和初始值相同

```cpp
auto n = 10;   //int
auto x = 1.5;  //double
auto y = 1.3e12L //long double

auto a = 0; //int，自动类型判断不会判断double类型
auto b = 0.0;  //double
double c = 0;  //double，显示声明不会出现问题

//标准模块库(STL)的自动类型判断（复杂类型）
std::vector<double> scores;
//(1)
std::vector<double>::iterator pv = scores.begin();
//(2)
auto pv = scores.begin();
```

***

# 5. 小结

> C++的基本类型分为两组：  
>> 一组存储整数的类型  
>> 一组存储浮点数的类型  
>
> 整型由小到大依次是：  
>> `bool, char, signed char, unsigned char, short, signed char, unsigned char, int, signed int, unsigned int, long, unsinged long, long long, unsigned long long`  
>>
>> 还有在一种`wchar_t`类型，取决于实现  
>> C++11新增的`char16_t`和`char32_t`，分别存储16和32位的字符编码  
>> `short` 至少16位  
>> `int` 至少和`short`一样长  
>> `long` 至少32位  
