---
layout: single
related: false
title: C++变量
date:   2019-07-21 21:11:47
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，变量的命名规则、类型概述

# 1. C++变量命名规则

+ 只能使用字母、数字、下划线  
+ 第一个字符不能是数字  
+ 区分大小写
+ 不能使用关键字
+ 以两个下划线或下划线和大写字母打头的名称保留给实现（编译器及其使用的资源）使用，以一个下划线开头的名称被保留给实现，用作全局标识符
+ C++对命名的长度没有限制，但是有些平台会限制长度

> 常用描述类型或者变量的命名方式，比如： str或者sz（表示以空字符结束的字符串）、b（表示布尔值）、p（表示指针）、c（表示单个字符）

# 2. 整型

```shell
C++的基本整型按照宽度递增（width,用于描述存储整型时候使用的内存量。内存越多，则越宽）的排序顺序分别是：char、short、int、long和C++11新增的long long。
```

# 3. short、int、long、long long

```shell
计算机内存的基本单元是bit位。
关表示0，开表示1.
8位的内存内存块可以设置256种不同的组合（2的八次方）。
因此，8位单元可以表示0-255或者-128到127。

字节byte通常表示8位的内存单元。从这个意义来说，
字节指的是描述计算机内存量的度量单位。
1KB=1024byte
1MB=1024KB

在美国，基本字符集通常是ASCII和EBCDIC字符集，他们可以用8位表示一个字节。
但是在国际编程中可能需要使用更大的字符集，如Unicode，因此有些实现可能使用16位甚至32位的字节。
```

当前很多系统都使用最小长度，即short是16位。  
这为int提供了多种选择，可以是16位、24位、32位。甚至64位。  
因为long和long long至少长64位。

short是short int的简称。long是long int的简称。

short、int、long、long long都是符号类型。这意味着每种类型的取值范围，负值和正值几乎相同。

__示例代码：__

```cpp
//limits.cpp
#include<iostream>
#include<climits> //use limits.h for older system

int main() {
        using namespace std;
        int n_int = INT_MAX;
        short n_short  = SHRT_MAX; //symbols defined in climts file
        long n_long = LONG_MAX;
        long long n_llong = LLONG_MAX;

        cout << "int is"<< sizeof(n_int) << " bytes" << endl;
        cout << "short is"<< sizeof(n_short) << " bytes" << endl;
        cout << "long is"<< sizeof(n_long) << " bytes" << endl;
        cout << "long long is"<< sizeof(n_llong) << " bytes" << endl;
        cout << endl;

        cout << "Maximum value:"<< endl;
        cout << "int : " << n_int << endl;
        cout << "short : " << n_short << endl;
        cout << "long : " << n_long << endl;
        cout << "long long : " << n_llong << endl;

        cout << "Minimum int value = "<<INT_MIN<<endl;
        cout << "BNits per bytr = "<< CHAR_BIT <<endl;
        return 0;
}
```

__执行结果：__

```cpp
//Results
int is 4 bytes
short is 2 bytes
long is 8 bytes
long long is 8 bytes

Maximum value:
int : 2147483647
short : 32767
long : 9223372036854775807
long long : 9223372036854775807
Minimum int value = -2147483648
BNits per bytr = 8
```

## 3.1. climits文件的符号常量

符号常量|极值意义
:-:|:-:
CHAR_INT|char的位数
CHAR_MAX|char的最大值
CHAR_INT|char的最小值
SCHAR_MAX|signed char的最大值
UCHAR_MAX|unsigned char的最大值
SHRT_MAX|short的最大值
USHRT_MAX|unsigned short的最大值
INT_MAX|int的最大值
LONG_MAX|long的最大值
LLONG_MAX|long long的最大值

# 4. 初始化

```cpp
int n_int = INT_MAX;  

short year;  
year = 1492;  

//C++11的初始化方式
int hamburgers = {24};  //set hanmburgers to 24

int emus{7};  //设置emus为7
int rheas = {12};  //设置rheas为12

int rocs = {};  //设置为0
int rocs{};
```

# 5. 无符号类型

```cpp
例如：
short的unsigned无符号的表示范围是0-65535
short的表示范围是-32768到+32767
```

# 6. C++如何确定常量的类型

```shell
例如：

`cout << "year" << 2019 << endl;`

程序会把2019存储为int类型。

整型后面的l或者L后缀表示为long；

u或者U表示unsigned int常量；

ul或者UL表示unsigned long；

C++11还提供了ull, ULL, uLL和Ull。

```

***

# 7. char字符型

> char是专门存储字符（字母或者数字）的类型。

```cpp
//testChar
char ch = 'M';

cout.put(ch);
```

## 7.1. char的字面值

```shell
ASCII系统中对应的数值编码：
'A' = 65
'a' = 97
'5' = 53
'' = 32  //空格字
'!' = 33
```
