---
layout: single
related: false
title:  C++函数模块（函数指针、递归）
date:   2019-08-26 22:23:05
categories: cpp
tags: cpp
toc: true
---


> C++基础语法，包含C++内联函数、引用变量、函数重载、函数模板

# 1. 内联函数

> 常规函数调用使程序跳到另一个地址（函数的地址），并在函数结束后返回。来回跳跃并记录跳跃位置意味着一定的开销。  
> 内联函数使得编译器将相应的函数代码替换函数调用。程序无需跳到另一个位置处执行代码，再调回来。  
> 因而，内联函数运行速度比常规函数快，但是代价是需要占用更多的内存。如果程序在10个不同的地方调用调用同一个内联函数，则该程序将包含该函数代码的10个副本。

使用内联函数：
+ 在函数声明前加上关键字`inline`
+ 在函数定义前加上关键字`inline`

通常的做法是省略原型，将整个定义（即函数头和所有函数代码）放在本应提供原型的地方。

**示例代码：**

```cpp
// inline.cpp 
#include<iostream>
using namespace std;

inline double square(double x) {return x * x;}

int main() {
	double a, b;

	a = square(5.0);
	b = square(4.5 + 7.5);

	cout << "a = " << a << ", b = " << b <<endl;
	return 0;
}
```

**执行结果：** `a = 25, b = 144`

**NOTE：**  
C语言使用预处理语句`**#define`提供宏（内联代码的原始实现），比如：`#define SQUARE(X) X*X`  
这是通过文本替换来实现的。

***
# 2. 引用变量`标识符&`

> 引用变量的主要用途是用作函数的形参。通过将引用变量用作参数，函数将使用原始数据，而不是副本。这样除了指针以外，引用也为函数处理大型结构提供了方便的途径。

## 2.1. 创建引用变量

```cpp
int rats;
int & rodents = rats;  //此处的&不是地址运算符，而是类型标识符的一部分。int&是指向int的引用。
```

上述的引用声明允许将rats和rodents互换，`他们指向相同的值和内存单元`。
**示例代码：**

```cpp
// firstref.cpp 
#include<iostream>

using namespace std;

int main(){
	int rats = 101;
	int & rodents = rats;   //必须在声明引用时初始化，不可以分成两句
    //上一句等价于  int * const pr = &rats;

	cout << "rats = " << rats << ", rodents = " << rodents <<endl;

	rodents++;
	cout << "After rodents++, rats = " << rats << ", rodents = " << rodents << endl;
	cout << "Address &rats = " << &rats << ", &rodents = " << &rodents <<endl;
	return 0;
}
```

**执行结果：**

```cpp
rats = 101, rodents = 101
After rodents++, rats = 102, rodents = 102
Address &rats = 0x7ffe8c156f1c, &rodents = 0x7ffe8c156f1c  //相同的地址和值
```

## 2.2. 将引用用作函数参数

```
//按值传递
void lazy(int x);
int main() {
    int times = 20;
    lazy(times);
    return 0;
}

void lazy(int x) {
    ...
}

//按引用传递
void work(int &x);
int main() {
    int times = 20;
    work(times);
}

void work(int &x) {
    ...
}
```

如果让函数使用传递给她的信息，而不对信息进行修改，同时又箱使用引用，则应使用`常量引用`，即使用const。

`double refcube(const double &ra);`

将引用参数声明为常量数据的引用的理由三个：

+ 使用const可以避免无意中修改数据的编程错误
+ 使用const使函数能够处理const和非const实参，否则将只能接受非cosnt数据
+ 使用const引用使函数能够正确生成并使用临时变量

C++11新增了另一种引用 -- `右值引用`，这种引用可以指向右值，是使用`&&`声明的。目的用来实现移动语义。而'&'引用是左值引用。

```cpp
double && rref = std::sqrt(36.00);  //6
double j = 15.0;
douvle && jref = 2.0 * j + 18.5;   //48.5
```

## 2.3. 合适使用引用参数

使用引用参数的主要原因有两个：
+ 能够修改调用函数中的数据对象
+ 通过传递引用而不是整个数据对象，提高程序运行速度

对于使用传递的值而不做修改的函数：
+ 如果数据对象很小，如内置数据类型或小型数据。则按值传递
+ 如果数据对象是数组，则使用指针，并将指针声明为指向const的指针
+ 如果数据对象是较大的结构，则使用const指针或const引用，提高效率，节省复制结构所需的时间和空间
+ 如果数据对象是类对象，则使用const引用，传递类对象参数的标准方式是按引用传递

对于修改函数中数据的函数：
+ 如果数据对象是内置数据类型，则使用指针。例如调用fixit(&x)这样的函数，则很明显要修改x
+ 如果数据对象是数组，则只能使用指针
+ 如果数据对象是结构，则使用引用或者指针
+ 如果数据对象是类对象，则使用引用

# 3. 默认参数

> `默认参数`指的是当函数调用中省略了实参时自动调用的一个值。例如将`void wow(int n)`设置成n有默认值是1，则函数调用`wow()`就等价于`wow(1)`

通过函数原型设置默认值：`char * left(const char *str, int n = 1);`

***

# 4. 函数重载

> 同名的函数使用不同的参数列表。  
> 因而关键是参数列表，也称作`函数特征标`。如果两个函数的参数数目和类型相同，同时参数的排列顺序相同，则他们的`特征标相同`。  
> `返回类型可以不同，但是特征标也必须不同`

```cpp
void print(const char *str, int width);
void print(double d, int width);
void print(long l, int width);
void print(int i, int width);
void print(const char *str);

//返回类型不同
long gronk(int n, float m);

double gronk(int n, float);   //互斥，不允许重载
double gronk(float n, float m);   //可以重载
```

**因而在使用`print()`函数时，编译器根据采取的用法使用有相应特征标的函数原型。**

```cpp
print(1999.0, 10);
print(1999L, 15);
print("pen", 12);
```

当函数基本上执行相同的人物，但使用不同形式的数据时，才应采用函数重载。

## 4.1. 名称修饰（或名称矫正）

根据函数原型中指定的形参类型对每个函数名进行加密。  
`long MyFun(int, float);`  
编译器将其转化为内部表示来描述接口：  
`?MyFun@@YAXH`

***

# 5. 函数模板

> 函数模板是通用的函数描述，使用泛型来定义函数，其中泛型可以用具体的类型（如int或double）替换。  
> 通过将类型作为参数传递给模板，可以使编译器生成该类型的函数  
> 由于模板允许以泛型的方式编写程序，因此也称作`通用编程`  
> 由于类型是用参数表示的，因此模板特性有时也称作`参数化类型`

函数模板允许以任意类型的方式来定义函数。例如，可以建立一个交换模板：

```cpp
template <typename AnyType>
//template <class T>

void swap(AnyType &a, AnyType &b) {
	AnyType temp;
	temp = a;
	a = b;
	b = temp;
}
```

第一行建立一个模板，将类型命名为AnyType，关键字`template`和`typename`是必需的，除非使用另一个关键字`class`代替`typename`(这两个关键字等价的)。另外，必须使用尖括号。  
类型名可以任意选择（此处是AnyType），常用`T`

> 如果需要多个将同一个算法用于不同类型的函数，请使用函数模板。如果不考虑向后兼容的问题，并愿意键入较长的单词，则声明类型参数时，应使用关键字typename而不使用class。

**Example:**

```cpp
// funtemp.cpp 
#include<iostream>

using namespace std;

template <typename T> //or class T
void Swap(T &a, T &b);  //函数原型、函数模板使用、引用变量

//函数模板定义
template <typename T> //or class T
void Swap(T &a, T &b) {
	T temp;
	temp = a;
	a = b;
	b = temp;
}

int main(){
	int i = 10;
	int j = 20;
	cout << "Before swap, i = " << i << ",j = " << j <<endl;
	Swap(i, j);
	cout << "After  swap, i = " << i << ",j = " << j <<endl;

	double x = 24.45;
	double y = 45.2;
	cout << "Before swap, x = " << x << ",y = " << y <<endl;
        Swap(x, y);
        cout << "After  swap, x = " << x << ",y = " << y <<endl;

	return 0;
}
```

**Result:**

```cpp
Before swap, i = 10,j = 20   //当接收两个int参数，则会用int代替T
After  swap, i = 20,j = 10
Before swap, x = 24.45,y = 45.2
After  swap, x = 45.2,y = 24.45
```

**Note:**  
函数模板不能缩短可执行程序，最终仍将由独立的函数定义，最终的代码不包含任何函数模板，而只包含了为程序生成的实际函数。

使用函数模板的好处是使得多个函数定义更简单可靠。

***

## 5.1. 重载的函数模板

> 可以像重载常规函数定义一样重载函数模板定义。保证被重载的函数模板`特征标`必须不同

```cpp
template <typename T>
void Swap(T &a, T &b);

template <typename T>
void Swap(T *a, T *b, int n);

//实现第二个
template <typename T>
void Swap(T a[], T b[], int n) {
	T temp;
	for (int i = 0; i < n; i++) {
		temp = a[i];
		a[i] = b[i];
		b[i] = temp;
	}
}
```

## 5.2. 局限性

> 模板函数可能无法处理某些类型。例如数组、指针、结构的某些运算。  
> 一种解决方案是C++允许你重载运算符+，以便能够将其用于特定的结构或类；  
> 另一种是为特定类型提供具体化的模板定义。

## 5.3. 显示具体化

C++98标准选择了以下的方法实现第三代具体化：
+ 对于给定的函数名，可以有非模板函数、模板函数、显示具体化模板函数、以及他们的重载版本
+ 显示具体化的原型和定义应该以`template <>`开头，并通过名称指出类型
+ 优先级： 非模板函数 > 显示具体化模板函数 > 常规模板函数

```cpp
struct job {
	char name[40];
	double salary;
	int floor;
};

//非模板函数(首先调用)
void Swap(job &, job &);

//显示具体化模板函数（其次调用）
template <> void Swap<job>(job & ,job &);

//常规模板函数
template <typename T>
void Swap(T &, T &);
```

`Swap<job>`的`<job>`是可选的，因为函数的参数类型声明，这是job的一个具体化。因此，该原型也可以写作：  
`template <> void Swap(job &, job &);`

**Example:**

```cpp
// twoswap.cpp 
#include<iostream>
using namespace std;

template <typename T>
void Swap(T &a, T &b);

struct job{
	char name[40];
	double salary;
	int floor;
};

template <> void Swap<job>(job &j1, job &j2);

void Show(job &j);

template <typename T>
void Swap(T &a, T &b) {
	T temp;
	temp = a;
	a = b;
	b = temp;
}

template <> void Swap<job>(job &j1, job &j2) {
	double t1;
	int t2;
	t1 = j1.salary;
	j1.salary = j2.salary;
	j2.salary = t1;

	t2 = j1.floor;
	j1.floor = j2.floor;
	j2.floor = t2;	
}

void Show(job &j) {
	cout << j.name << " : $" << j.salary << " on floor " << j.floor <<endl;
}

int main(){
	cout.precision(2);
	cout.setf(ios::fixed, ios::floatfield);

	int i = 10, j = 20;
	cout << "Before swap, i = " << i << ",j = " << j <<endl;
	Swap(i, j);
	cout << "After  swap, i = " << i << ",j = " << j <<endl;

	job sue = {"Susan", 7300.60, 7};
	job sidney = {"Sidney Taffee", 78060.72, 9};
	cout << "Before job swap,\n";
	Show(sue);
	Show(sidney);
	Swap(sue, sidney);
	cout << "After  job swap,\n";
        Show(sue);
        Show(sidney);

	return 0;
}
```

**Results:**

```cpp
Before swap, i = 10,j = 20
After  swap, i = 20,j = 10
Before job swap,
Susan : $7300.60 on floor 7
Sidney Taffee : $78060.72 on floor 9
After  job swap,
Susan : $78060.72 on floor 9
Sidney Taffee : $7300.60 on floor 7
```

## 5.4. 关键字decltype(c++11)

```cpp
template <class T1, class T2>
void ft(T1 x, T2 y) {
	...
	?type? xpy = x + y;
	...
}
```

此处的xpy不知道如何确定类型，在C++11中提供关键字`decltype`，使用方法：

```cpp
int x;
decltype(x) y;

decltype(x+y) xpy;
xpy = x + y;

//或者合并成一句
decltype(x+y) xpy = x + y;

//举例
decltype(auto) a;

//完善上述的模板函数
template <class T1, class T2>
void ft(T1 x, T2 y) {
	...
	decltype(x+y) xpy = x + y;
	...
}
```

## 5.5. C++11后置返回类型声明语法

```cpp
template<class T1, class T2>
?type? gt(T1 x, T2 y) {
	return x + y;
}
```

此处无法确定返回的类型，因为未声明参数x和y，所以他们不再作用域呢，无法使用decltype关键字（必须声明参数后使用）。C++11新增了一种语法：

`double h(int x, float y);`  
使用新增的语法后可以这样编写：

`auto h(int x, float y) -> double;`

这样将返回类型已到了参数声明之后，`->double`被称为`后置返回类型`。

因而使用这种方法声明模板函数：

```cpp
template<class T1, class T2>
auto gt(T1 x, T2 y) -> decltype(x+y) {
	return x + y;
}
```
