---
layout: single
related: false
title:  C++头文件、作用域、内存模型和名称空间
date:   2019-09-01 21:52:00
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，包含C++头文件、作用域、内存模型和名称空间

# 1. 单独编译

C++提供`#include`语法，因而可以将程序划分，大致可以分成三部分：  
1. 头文件`a.h`：包含结构声明和使用这些结构的函数原型
2. 源代码文件`a.cpp`：包含与结构相关的函数的代码
3. 源代码文件`b.cpp`：包含调用与结构相关的函数的代码

## 1.1. 头文件内容和引用

> 头文件常包含的内容：
+ 函数原型
+ 使用`#define`或者`const`定义的符号常量
+ 结构声明struct
+ 类声明class
+ 模板声明template
+ 内联函数inline

> 头文件引用时，使用`#include "coordin.h"`，而不是`#include <coordin.h>`  
> 因为尖括号的文件名，C++编译器将在存储标准头文件的主机系统的文件系统中查找；而双引号的头文件，则编译器将首先查找当前的工作目录或源代码目录（或其他目录，取决于编译器）

## 1.2. 头文件管理

在同一个文件中只能将同一个头文件包含一次。有时候会存在包含了另一个头文件的头文件。因而有一种方法基于预处理器编译指令`#ifndef即if not defined`可以忽略第一次包含之外的所有内容，但是这种方法不能防止编译器将文件包含两次。。

```c++
//仅当以前没有使用预处理器编译指令#define定义名称COORDIN_H_时，才处理#inndef和#endif之间的语句
#ifndef COORDIN_H_
#define COORDIN_H_  //完成该名称的定义
...
#endif
```

***

# 2. 作用域`scope`

> 作用域描述名称在文件的多大范围可见。  
> C++变量的作用域有多种。
+ 局部变量只在定义他的代码块中可用。
+ 作用域为全局（也叫文件作用域）的变量在定义位置到文件结尾之间都可用。
+ 自动变量的作用域为局部
+ 静态变量的作用域是全局还是局部取决于它如何被定义
+ 在函数原型作用域中使用的名称只包含参数列表的括号内可用
+ 在类中声明的成员的作用域为整个类
+ 在名称空间中声明的变量的作用域是整个名称空间
+ C++函数的作用域可以是整个类或者命名空间，但不能是局部的，不能只对自己可见，这样会导致不能被其他函数调用

不同的C++存储方式是通过存储持续性（数据内存存储方式）、作用域和链接性来描述的。

# 3. 数据内存的存储方式（存储持续性）

> C++使用三种方案存储数据，见文章《C++内存分配方式和模板类vector,array》  
> C++11新增了`线程存储持续性`

## 3.1. 线程存储持续性

在多核处理器很常见，这些CPU可以同时处理多个执行任务（可以使用SDK的systrace工具抓取一份trace查看）。这让程序能够将计算放在可并行处理的不同线程中。如果变量是使用关键字`thread_local`声明的，则其生命周期和所属的线程一样长。

## 3.2. 自动存储持续性

> 默认情况下，在函数中声明的函数参数和变量的存储持续性是自动，作用域是局部，没有链接性。当函数结束时，这些变量将小时（执行到代码块时，将为变量分配内存，但是其作用域的起点是其声明位置）

如果在代码块中定义了变量，则该变量的存在时间和作用域将被限制在该代码块内。

通常存储在栈stack中，先进后出。之所以被称为栈，是由于新数据被象征性的放在原有数据的上面（相邻的内存单元），当程序使用完后，将其从栈中删除（栈的默认长度取决于实现，编译器通常提供改变栈长度的选项）。程序使用两个指针跟踪栈，一个指向栈底（开始位置），一个指向栈顶（下一个可用内存单元）。当函数被调用时，其自动变量被加入栈中，栈顶指针指向变量后的下一个可用内存单元。函数结束时，栈顶指针被重置为函数被调用前的值，从而释放新变量使用的内存。

有些自动变量存储在**寄存器**中，关键字`register`最初由C语言引入，C++11之前，它建议编译器使用CPU寄存器存储自动变量。这旨在提高访问变量的速度。  
`register int count_fast;`

在C++11中，失去了这种提示作用，关键字`register`只是显式的指出变量是自动的。鉴于它只能用于原本就是自动的变量，使用他的唯一原因是指出这个变量的名称可能与外部变量相同，避免使用了该关键字的现有代码非法。

## 3.3. 静态持续变量

C++未静态存储持续性变量提供三种链接性：
1. 外部链接性（可在其他文件中访问）
2. 内部链接性（只能在当前文件中访问）
3. 无链接性（只能在当前函数或者代码块中访问）

```c++
...
int global = 1000;   //外部链接，作用域是整个文件
static int one_file = 50; //内部链接，作用域是整个文件
int main(){
  ...
}

void fun1(int n) {
  static int count = 0; //无链接
  int ma = 0;   //自动变量，两者区别在于静态无链接变量在函数没有被执行时，也会在内存中
  ...
}
```

存储描述|持续性|作用域|链接性|声明方法
:-:|:-:|:-:|:-:|:-:
自动| 自动 |代码块 |无|代码块中
寄存器| 自动 | 代码块 |无|代码块中，使用关键字`register`
静态，无链接性| 静态 | 代码块 | 无|使用关键字'static'
静态，外部链接性| 静态 |文件|外部|不再任何函数内
静态，内部链接性| 静态| 文件|内部|不再任何函数内，使用关键字`static`

所有静态持续变量的初始化特征：**未被出的初始化的静态变量的所在位都被设置为0，这种变量被称为零初始化的**

零初始化和常量表达式初始化被统称为静态初始化，这意味着在编译器处理文件（翻译单元）时初始化变量。动态初始化意味着变量将在编译后初始化。

```c++
int x;   //零初始化
int y = 5; //y先被零初始化，然后编译器计算常量表达式，初始化成5
```

***

### 3.3.1. 静态持续性、外部链接性

链接性为外部的变量通常简称为外部变量，存储持续性是静态，作用域是整个文件。在函数外部定义，因此对所有函数都是外部的。例如可以在main()前面或者头文件中定义他们。可以在文件中位于外部变量定义后面的任何函数中使用它，因此也被称为`全局变量`（相对于局部的自动变量）

C++有“单定义规则”，每个变量只能有一次定义。因而C++提供两种变量声明，一种是`定义声明`，分配存储空间；另一种是`引用声明`，不给变量分配存储空间，引用已有的变量。

引用声明使用关键字extern，则不进行初始化。

```c++
double up;  //零初始化
extern int blem;;  //引用声明，不进行初始化（此处可以引用其他文件已经声明的外部变量）
extern char gr = 'z';  //已经初始化了，所以导致分配存储空间
```

### 3.3.2. 静态持续性、内部链接性

不同于外部变量，将static限定符用于作用域为整个文件的变量时，该变量的链接性是内部的。**两者区别是** 内部链接的变量只能在所属的文件中使用，外部变量具有外部链接性，可以在其他文件中使用。

### 3.3.3. 静态持续性、无链接性

将static限定符用于在代码块中定义的变量，导致局部变量的存储持续性是静态的。这意味着虽然只能在该代码块使用，但是在该代码块不处于活动状态时仍然存在。`因此在两次调用该函数时，静态局部变量的值将保持不变。此外如果初始化了静态局部变量，则程序只在启动时进行一次初始化，再次调用时不会初始化，即值保持上次的值`

***

# 4. 限定符和说明符

## 4.1. 存储说明符

+ auto(在C++11不再是说明符)
+ register
+ static
+ extern
+ thread_local
+ mutable

1. 同一个声明中不能使用多个说明符（除了thread_local可以和static或者extern结合使用）
2. C++11之前auto指出变量是自动变量，但是在C++11中，auto用于自动类型推断
3. register用于在声明中指示寄存器存储，在C++11中只是显式的指出变量是自动的
4. static被用在作用域是整个文件的声明中时，表示内部链接性；被用于局部声明中，表示局部变量的存储持续性是静态的
5. extern表明是引用声明，即声明引用在其他地方定义的变量
6. thread_local指出变量的持续性和所属线程的持续性相同。thread_local变量之于线程，犹如常规静态变量之于整个程序。
7. mutable的含义根据const来解释（`查看下一小节`）

## 4.2. cv-限定符(const和volatile)

+ const
+ volatile

1. const表示内存被初始化后，程序不能再对其进行修改
2. volatile表明即使程序代码没有对内存单元进行修改，其值也可能发生变化。作用是为了改善编译器的优化能力。例如，如果编译器发现程序在几条语句中两次使用了某个变量的值，则编译器可能不是让程序查找这个值两次，而是将这个值缓存到既存区中。（这种优化假设变量的值在两次使用之间不发生变化）将变量声明为volatile，则编译器不会进行这种优化；否则将进行这种优化。
3. 可以使用`mutable`指出，即使结构（或者类）变量为const，其某个成员也可以被修改。

```c++
struct data {
  char name[30];
  mutable int accesses;
  ...
};

const data veep = {"peter", 0, ...};
+ strcpy(veep.name, "nancy");  //not allowed,因为const
- veep.accesses++;  //allow
```

veep的const限定符禁止程序修改veep的成员，但access成员的mutable说明符可以使的access不受这种限制。

***

# 5. 名称空间

1. 声明区域是可以在其中进行声明的区域。例如函数外面声明全局变量，其声明区域为所在的文件；
函数中声明变量，声明区域为所在代码块；
2. 潜在作用域：变量的潜在作用域从声明点开始，到其声明区域的结尾。因此潜在作用域比声明作用域小，这是由于变量必须定义后才能使用。
3. 关键字`namespace`通过定义一种新的声明区域来创建命名的空间，如此不会和另一个名称空间发生同名冲突。

```c++
namespace Javk {
  double pail;
  int pal;
  ...
}

namespace Jil {
  double pail;
  int pal;
  ...
}
```

## 5.1. 作用域解析运算符`::双冒号`

```c++
Jil::pal = 3;   //限定的名称
int pal = 10;   //未限定的名称
Javk::pail = 23.4;
```

## 5.2. using声明恶化using编译指令

> using声明使得特定的标识符可用，using编译指令使得整个命名空间可用。

```c++
namespace Jil {
  ...
}

int main() {
  using Jil::pail;  //using声明
  double a;
}
```

using编译指令由名称空间和关键字`using namespace`组成。

```c++
using namespace Jil;
using namespace std;
```

## 5.3. 建议

+ 不要在头文件使用using编译指令，如果非要使用，应将其放在所有的预处理编译指令`#include`后面
+ 导入名称时，首选使用作用域解析运算符或者using声明的方法
+ 对于using声明，首选将其作用域设置为局部而不是全局