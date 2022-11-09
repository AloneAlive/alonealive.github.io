---
layout: single
related: false
title:  C++对象和类
date:   2019-09-04 22:52:00
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，包含C++对象和类

# 1. 类型

指定基本类型完成了三项工作：
1. 决定数据对象需要的内存数量
2. 决定如何解释内存中的位（long和float所占位数相同，但是将他们转换成数值的方法不同）
3. 决定可使用数据对象执行的操作和方法

# 2. 类class

类规范由两部分组成：
1. 类声明：以数据成员的方式描述数据部分，以成员函数（方法）的方式描述公有接口
2. 类方法定义：描述如何实现类成员函数

即类声明提供类的蓝图，方法定义则提供了细节。

类声明举例：

```c++
// stock00.h
#ifndef STOCKOO_H_
#define STOCKOO_H_

#include<string>

class Stock {
    private:  //只能通过公共成员（或友元函数）访问的类成员，例如要修改shares，只能通过Stock的成员函数
        std::string company;
        long shares;
        double share_val;
        double total_val;
        void set_tot() {
            total_val = shares * share_val;
        }
    public:
        void acquire(const std::String &co, long n);  //函数原型
    protected:
        ...
}
```

## 2.1. 实现类成员函数

> 为由类声明中的函数原型表示的成员函数提供代码。成员函数定义和常规函数相似，有函数头和函数体、返回类型和参数，但是有两个特征：
1.定义成员函数时，使用作用域解析运算符`::`来标识函数所属的类

`void Stock::update(double price) {...}`

此处的update即是Stock的成员函数，这就意味着可以将另一个类的成员函数也命名为update。

2.类方法可以访问类的private组件

## 2.2. 构造函数

> 函数声明对象时，将自动调用构造函数。构造函数的参数表示的不是类成员，而是赋给类成员的值。因此，参数名不能和类成员相同，否则会导致混乱。

**使用构造函数：**

显示调用构造函数：
`Stock food = Stock("World cabbage", 250, 1.25);`

隐式调用构造函数：
`Stock garment("Furry", 50, 2.6);`

创建类对象：
`Stock *pstock = new Stock("ABC", 18, 19.0);`

**Notes:**

如果没有提供任何构造函数，则自动提供默认构造函数。
比如：`Sotck::stock() {...}`

如果定义了构造函数，就必须提供默认构造函数，否则会报错。定义的方式有两种：
1. 给已有构造函数的所有参数提供默认值，`Stock(const string&co = "Error", int n =0;)`
2. 通过函数重载来定义另一个没有参数的构造函数，`Stock()`

而只能拥有一个默认构造函数，所以不要同时使用这两个方式。而用户定义的默认构造函数通常给所有成员提供了隐式初始值。例如：

```c++
Stock::Stock() {
  company = "no name";
  share = 0;
  share_val = 0.0;
  total_val = 0.0;
}
```

隐式的调用默认构造函数时，不用使用圆括号。`Stock stock1;`

## 2.3. 析构函数

> 如果构造函数使用new发根配内存，则析构函数将使用delete释放内存。而析构函数的名称是在函数名前加上`~`，例如`~Stock()`。Stock的析构函数不承担任何重要的工作，因此直接编写不执行任何操作的函数：

```c++
Stock::~Stock() {

}
```

## 2.4. 编译器决定调用析构函数的时机

+ 如果创建的是静态存储对象，则其析构函数就爱你挂在程序结束时自动调用
+ 如果创建的是自动存储类对象，则其析构函数将在程序执行完代码块时（该对象是在其中定义的）自动被调用
+ 如果对象通过new创建的，则将驻留在栈内存或者自由存储区中，当使用delete来释放内存时，其析构函数将自动被调用
+ 程序可以创建临时对象来完成特定的操作，此时程序将在结束对该对象使用时自动调用其析构函数

***

# 3. this指针

> this指针指向用来调用成员函数的对象（this被称为隐藏参数传递给方法）每个成员函数（包含构造函数和析构函数）都有一个this指针。this指针指向调用对象。如果方法想要引用整个调用对象，则可以使用表达式`*this`。在函数的括号后面使用const限定符将this限定为const，这样将不能使用this来修改对象的值。  
> 然而要返回的不是this，因为this是对象的地址，*this是指向的值。

```c++
const Stock & Stock::topval(const Stock & s) const {
  if (s.total_val > total_val)
    return s;
  else
    return *this;
}
```

# 4. 对象数组

创建同一个类的多个对象，创建对象数组比独立对象变量更合适。

`Stock mystuff[4];`

```c++
mystuff[0].update();
mystuff[3].show();
const Stock * tops = mystuff[2].topval(mystuff[1]);

//使用构造函数初始化数组元素，此时必须为没够元素调用构造函数
const int STKS = 4;
Stock stocks[STKS] = {
  Stock("A", 12.5, 20),
  Stock("B", 11.5, 530),
  Stock("C", 13.5, 120),
  Stock("D", 14.5, 30),
};
```

# 5. 类作用域

> 在类中定义的名称（如类数据称源和类成员函数名）的作用域都为整个类。在类声明或成员函数定义中，可以使用未修饰的成员名称（未限定的名称）。构造函数名称被调用时，才能被识别，因为他的名称和类名相同。在其他情况下，使用类成员名时，必须根据上下文使用直接成员运算符（.），间接成员运算符（->）或者作用域解析运算符（::）

## 5.1. 作用域于为类的常量

声明类只是描述了对象的形式，并没有创建对象。因此，在创建对象前，没有用于存储值的空间（C++11提供了成员初始化，但不适用于数组声明）。因此以下的方式初始化不正确：

```diff
class Bck {
  private:
- const int Months = 12; //fail
- double costs[Months];
}
```

可以使用以下方式,在类中声明一个枚举，枚举的作用域是整个类。

```diff
class Bck {
  private:
+ enum {Months= 12};
+ double costs[Months];
}
```

可以在类中定义常量的方式，使用关键字static。这将常量和其他静态变量存储在一起，而不是存储在对象中。因此只有一个Months常量，被所有的Bck类对象共享。

```diff
class Bck {
  private:
+ static const int Months= 12;
+ double costs[Months];
}
```

# 6. C++11作用域内枚举

传统的枚举容易出现冲突，例如在一个类中定义：
```c++
enum egg {Small. Large, Medium};
enum t_shirt {Small. Large, Medium};
```

C++11提供一种新枚举，其枚举量的作用域是类，如下：

```c++
enum class egg {Small. Large, Medium};
enum class t_shirt {Small. Large, Medium};
//或者使用struct关键字代替class
enum struct egg {Small. Large, Medium};
enum struct t_shirt {Small. Large, Medium};

egg choic = egg::Large;
t_shirt Floyd = t_shirt::Large;  //此时将不再发生冲突
```

# 7. 抽象数据类型

> C++使用栈来管理自动变量，当新的自动变量被生成后，他们被添加到栈顶；消亡时，从栈中删除他们。  

## 7.1. 栈的特征

栈存储了多个数据项（该特征使得栈成为一个容器，一种通用的抽象），其次，栈由可对他执行的操作来描述。
+ 可创建空栈
+ 可将数据项添加到栈顶（压入）
+ 可从栈顶删除数据项（弹出）
+ 可查看栈是否填满
+ 可查看栈是否为空

如果将上述描述转换为一个类声明，其中公有成员函数提供了栈操作的接口，而私有数据成员负责存储栈数据。私有部分必须表明数据存储的方式，例如可以使用常规数组、动态分配数组或者更高级的数据结构。

# 8. 小结

1. 通常将类声明分成两部分，类声明（包含函数原型表示的方法）应该放在头文件中。定义成员函数的源代码放在方法文件中。
2. 使用OOP方法的第一步是根据他和程序之间的接口描述数据，从而指定如何使用数据。然后设计一个类来实现该接口。一般来说，私有数据成员存储信息，公有成员函数（又称作方法）提供访问数据的唯一途径。类将数据和方法组合成一个单元，其私有性实现数据隐藏。
3. 类是用户定义的类型，而对象是类的实例。也可以说对象是这种类型的变量，例如由new按类描述分配的内存。
4. 如果希望成员函数对多个对象进行操作，可以将额外的对象作为参数传递给它。如果方法需要显示的引用调用他的对象，则可以使用this指针。由于this指针被设置为调用对象的地址，因此*this是该对象的别名。