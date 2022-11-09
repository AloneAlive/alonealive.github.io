---
layout: single
related: false
title:  C++运算符重载、友元、返回对象
date:   2019-09-16 22:59:00
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，包含C++运算符重载、友元、返回对象

# 1. 运算符重载

运算符重载将重载的概念扩展到运算符上，允许赋予C++运算符多种含义。

例如`*` 用于地址，将获得存储在这个地址中的值；而用于两个数字之间，得到的是乘积。

要重载运算符，可以使用被称为运算符函数的特殊函数形式，运算符函数的格式如下：
`operator(argument-list)`

例如`operator+()`重载`+`运算符，`operator*()`重载*运算符。op必须是有效的运算符，不能虚构新的符号。比如不存在`operator@()`。

```cpp
Time Time::operator+(const Time &t) const {
    Time sum;
    sum.minutes = minutes + t.minutes;
    sum.hours = hours + t.hours + sum.minutes / 60;
    sum.minutes %= 60;
    return sum;
}

```

## 1.1. 重载限制

1.重载后的运算符必须至少有一个操作数是用户定义的类型。防止用户为标准类型重载运算符。即不能用减法运算符重载求和，会影响性能。
2.使用运算符不能违反运算符原来的句法规则。例如不能将求模（余数）运算符重载成使用一个操作数：

```diff
- int x;
- % x;  //Error
```

同时不能修改运算符的优先级。

3.不能创建新运算符，例如不能定义`operator**()`函数来表示求幂。
4.**不能重载以下运算符：**

运算符|释义
:-:|:-:
sizeof|sizeof求长度运算符
.|成员运算符
.*|成员指针运算符
::|作用域解析运算符
?|条件运算符
typeid|一个RTTI运算符
const_cast|强制类型转换运算符
dynamic_cast|强制类型转换运算符
reinterpret_cast|强制类型转换运算符
static_cast|强制类型转换运算符

***

# 2. 友元

> C++控制对类对象私有部分的访问，通常公有类提供唯一的访问途径，但是有时候限制太严格，以致于不适合特定的编程情形。因而C++提供另一种形式的访问权限：**友元** ， 有三种：
1. 友元函数
2. 友元类
3. 友元成员函数（通过让函数成为类的友元，可以赋予该函数与类的成员函数相同的访问权限。）

在为类重载二元运算符时常常需要友元。

## 2.1. 创建友元

将原型放在类声明中，并在原型声明前加上关键字`friend`。

`friend Time operator* (double m, const Time &t);//goes in class declaration`

表明两点：
1. 虽然是在类声明中声明，但不是成员函数，因此不能使用成员运算符调用（.或者->），不能使用`Time::`限定符
2. 虽然不是成员函数，但是和成员函数的访问权限相同

> 在函数实现中不要使用关键字friend

```cpp
Time operator*(double m, const Time &t) {
    ......
}
```

# 3. 返回对象相关

当成员函数或独立的函数返回对象时，有几种返回方式可供选择，`可以返回指向对象的引用、指向对象的const引用或const对象`。

## 3.1. 返回指向const对象的引用

使用const引用的常见原因是极高效率，但是对何时可以采用这种方式存在限制。如果函数返回（通过调用对象的方法或者将对象作为参数）传递给他的对象，可以通过返回引用来提高效率。

```cpp
Vector force1(50, 60);
Vector force2(10, 70);
Vector max;
max = Max(force1, force2);

//version 1  
//返回对象将调用复制构造函数
Vector Max(const Vector & v1, const Vector & v2) {
    if (v1.magval() > v2.magval())
        return v1;
    else
        return v2;
}
//version 2 
//返回引用不会调用构造函数，因而效率更高；
//其次引用指向的对象在调用函数时存在（此处的force1和force2在调用函数中定义，满足条件）；
//v1和v2都被声明为const引用，因此返回类型必须是const，这样才匹配
const Vector & Max(const Vector & v1, const Vector & v2) {
    if (v1.magval() > v2.magval())
        return v1;
    else
        return v2;
}

```

## 3.2. 返回指向非const对象的引用

两种常见的返回非const对象的情形：
1. 重载赋值运算符（旨在提高效率）
2. 重载与const一起使用的`<<`运算符（必须这么做）

```cpp
//operator=()的返回值用于连续赋值
String s1("Good stuff");
String s2,s3;
s3 = s2 = s1; //s2.operator=()的返回值被赋值给s3，通过使用引用可以避免该函数调用String的复制构造函数来创建一个新的String对象
```

在这个例子中，返回类型不是const，因为方法operator=()返回一个指向s2的引用，可以对其进行修改。

```cpp
String s1("Good");
cout << s1 << "is coming";
```

在这个例子中，返回类型必须是ostream& ，而不能仅是ostream，否则将调用ostream类的复制构造函数，而ostream没有共有的复制构造函数。

## 3.3. 返回对象

如果返回的对象是被调用函数中的局部变量，则不应使用引用方式返回它。因为在被调用函数执行完毕，局部对象将调用其析构函数。

***

# 4. 小结

## 4.1. 指针和对象使用

1. 使用常规表示法来声明指向已有的对象，`String * glamour;`
2. 可以将指针初始化为指向已有的对象，`String * first = &sayings[0];`
3. 可以使用new来初始化指针，这将会创建一个新的对象，`String * favorite = new String(sayings[choice]);`
4. 对类使用new将调用相应的构造函数来初始化新建的对象
5. 可以使用`->`运算符通过指针访问类的方法，`shortest->length()`
6. 可以对对象的指针应用解除引用运算符`*`来获得对象，'if (sayings[i] < *first>)'

## 4.2. 重载<<运算符

要重新定义<<运算符，以便将他和cout一起用来显示对象的内容，请定义下面的友元运算符函数：

```cpp
//c_name是类名，如果该类提供了能够返回所需内容的公有方法，则可在运算符函数中使用这些方法，这样编不用将他们设置为友元函数
ostream & operator<< (ostream &os, const c_name & obj) {
    os << ...; //display object contents
    return os;
}
```

## 4.3. 转换函数

要将单个值转换为类类型，需要创建原型如下所示的类构造函数：`c_name(type_name value)`

其中c_name是类名，type_name是要转换的类型的名称。

要将类转换为其他类型，需要创建类如下所示的类成员函数:`operator type_name();`

虽然该函数没有声明返回类型，但应返回所需类型的值。使用转换函数时，可以在声明构造函数中使用关键字`explicit`，放置它被用于隐式转换。

## 4.4. 其构造函数使用new的类

+ 对于指向的内存是由new分配的所有类成员，都应在类的析构函数中对其使用delete，该运算符将释放分配的内存
+ 如果析构函数通过对指针类成员使用delete来释放内存，则每个构造函数都应当使用new来初始化指针，或将其设置为空指针
+ 构造函数中要么使用new，要么使用new[]，不能混用。对应的析构函数使用delete，和delete[]
+ 应定义一个分配内存的复制构造函数，这样程序能够将类对象初始化为另一个类对象，通常的函数原型：`className(const className &)`
+ 应定一个重载赋值运算符的类成员函数，其函数定义如下:

```cpp
c_name & c_name::operator=(const c_name & cn) {
    if (this == &cn)
        return *this;
    delete [] c_pointer; //是c_name的类成员，是指向type_name的指针
    //set size number of type_name unite to be copied
    c_pointer = new type_name[size];
    //then copy data pointed to by cn.c_pointer to location pointed to by c_pointer
    ...
    return *this;
}
```
***

# 5. `Share`一个Ubuntu免费使用ultraEdit的方法

在bin目录下建立一个脚本，并且在打开软件前执行：

```shell uexClearCache.sh 
rm -rfd ~/.idm/uex  
rm -rf ~/.idm/*.spl  
rm -rf /tmp/*.spl
```
