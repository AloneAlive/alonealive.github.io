---
layout: single
related: false
title:  C++函数模块（函数指针、递归）
date:   2019-08-21 22:23:05
categories: cpp
tags: cpp
toc: true
---

> 本章节主要是围绕函数为圆心，涉及到字符串、指针、C++11特性auto、typedef。  

# 1. 函数function

1. 是否有返回值（void）
2. main()函数
3. 函数原型，`diybke vikyne = cube(side);`，函数原型是一条语句，必须以分号结束。一般可以放在头文件中（.h）。函数原型可以确保以下几点：
+ + 编译器正确处理函数返回值
+ + 编译器检查使用的参数数目是否正确
+ + 编译器检查使用的参数类型是否正确

4. 传递给函数的参数类型和数量（形参）
+ + 可以有多个参数，通过逗号分隔
+ + 参数的变量名可以和函数原型的不同，而且原型的变量名可以省略`void n_chars(char, int);`

**示例代码：**

```cpp
//twoarg.cpp 
#include<iostream>
using namespace std;

void n_chars(char, int);   //函数原型

int main(){
	int times;
	char ch;

	cout << "Enter a character: ";
	cin >> ch;
	while (ch != 'q') {
		cout << "Enter an integer: ";
		cin >> times;
		n_chars(ch, times);   //调用函数
		cout << "\nEnter another character or press the 'q' to quit: ";
		cin >> ch;
	}
	cout << "The value of times is " << times <<endl;
	return 0;
}

void n_chars(char c, int n) {
	while (n-- > 0)
		cout << c;
}
```

**执行结果：**

```cpp
Enter a character: a
Enter an integer: 3
aaa
Enter another character or press the 'q' to quit: b
Enter an integer: 5
bbbbb
Enter another character or press the 'q' to quit: q
The value of times is 5
```

## 1.1. 函数使用指针处理数组

> C++将数组名解释为其第一个元素的地址:`cookies == &cookies[0]`  
> 数组声明使用数组名来标记存储位置  
> 对数组名使用sizeof将得到数组的长度（以字节为单位）  
> 将地址运算符&用于数组名，将得到整个数组的地址  
> **在C++中，只有用于函数头或者函数原型中，`int *arr`才等价于`int arr[]`**  

`例如函数原型： int sum_arr(int *arr, int n)`

## 1.2. 指针和const

1. 让指针指向一个常量对象，这样可以防止使用该指针修改所指向的值
2. 将指针本身声明为常量，可以防止改变指针指向的位置

```cpp
int age = 39;
const int *pt = &age;   //并不意味着指向的值是常量，而是对于pt而言，这个值是常量。即可以通过age改变age的值，但是不能通过pt指针改变它
```

建议将指针参数声明为指向常量数据的指针有两条理由：  
1. 可以避免由于无意间修改数据而导致的编程错误  
2. 使用const使得函数能够处理const和非const的实参，否则将只能接收非cosnt数据

```cpp
int sloth = 3;
const int * ps = &sloth;  //不允许ps来修改sloth的值，但是允许将ps指向另一个位置，即ps不是const，*ps是const
int * const finger = &sloth;  //允许*finger来修改sloth的值，但是finger只能指向sloth，即finger是const，但是*finger不是const
const int * const strick = &sloth;  //strick和*strick都是const
```

## 1.3. 函数和字符串

C-风格字符串的表示方式有三种：  
1. char数组
2. 用引号括起的字符串常量（也称作字符串字面值）
3. 被设置为字符串的地址的char指针

```cpp
char ghost[15] = "galloping";
char * str = "galumphing";
int n1 = strlen(ghost);  //ghost是&ghost[0]
int n2 = strlen(str);  //指向char
int n3 = strelan("gambolling");  //字符串string地址
```

**示例代码：**
```cpp
// strgfun.cpp 
#include<iostream>
using namespace std;

unsigned int c_in_str(const char * str, char ch);   //只针对非负数

int main() {
	char mmm[15] = "minimum";
	char *wail = "ululate";

	unsigned int ms = c_in_str(mmm, 'm');
	unsigned int us = c_in_str(wail, 'u');
	cout << ms << " m characters in " << mmm <<endl;
	cout << us << " u characters in " << wail <<endl;
	return 0;
}

unsigned int c_in_str(const char * str, char ch) {
	unsigned int count = 0;

	while (*str) {
		if (*str == ch) count++;
		str++;
	}
	return count;
}
```

**执行结果：**

```cpp
3 m characters in minimum
2 u characters in ululate
```

## 1.4. 返回字符串的函数

> 函数无法返回一个字符串，但是可以返回字符串的地址，并且效率更高

**示例代码：**
```cpp
//strgback.cpp 
#include<iostream>

using namespace std;

char * buildstr(char c, int n);

int main() {
	int times;
	char ch;

	cout << "Enter a character: ";
	cin >> ch;

	cout << "Enter an integer: ";
	cin >> times;

	char *ps = buildstr(ch, times);

	cout << "Result is " << ps <<endl;

	delete []ps;
	return 0;
}

char* buildstr(char c, int n) {
	char * pstr = new char[n+1];

	pstr[n] = '\0';
	while (n-- > 0)
		pstr[n] = c;
	return pstr;
}
```

**执行结果：**

```cpp
Enter a character: s
Enter an integer: 10
Result is ssssssssss
```

## 1.5. 函数和结构

最直接的方式是像处理基本类型那样处理结构，将结构作为函数传递，并在需要时将结构作返回值使用。

**示例代码：**
```cpp
// travel.cpp 
#include<iostream>

using namespace std;

struct travel_time {
	int hours;
	int mins;
};

const int Mins_per_hr = 60;

travel_time sum(travel_time tl, travel_time t2);
void show_time(travel_time t);

travel_time sum(travel_time t1, travel_time t2) {
	travel_time total;

	total.mins = (t1.mins + t2.mins) % Mins_per_hr;
	total.hours = t1.hours + t2.hours + (t1.mins + t2.mins) / Mins_per_hr;
	return total;
}

void show_time(travel_time t) {
	cout << t.hours << " hours, " << t.mins << " minutes\n";	
}

int main() {
	travel_time day1 = {5, 45};
	travel_time day2 = {4, 55};

	travel_time trip= sum(day1, day2);
	show_time(trip);

	return 0;
}
```

**执行结果**

`10 hours, 40 minutes`

## 1.6. 函数和string对象

**代码示例：**

```cpp
// topfive.cpp 
#include<iostream>
#include<string>

using namespace std;

const int SIZE = 5;

void display(const string sa[], int n);

void display(const string sa[], int n) {
	for (int i =0; i < n; i++)
		cout << i+1 << " : " << sa[i] << endl;
}

int main(){
	string list[SIZE];
	cout << "Enter your " << SIZE << " favorite astronomical sights: \n";

	for (int i =0; i < SIZE;i++) {
		cout << i+1 << " : ";
		getline(cin, list[i]);
	}

	cout << "Your list : \n";
	display(list, SIZE);
	return 0;
}
```

**执行结果：**

```cpp
Enter your 5 favorite astronomical sights: 
1 : Peter
2 : Nancy
3 : Good
4 : Wizzie
5 : Jack
Your list :
1 : Peter
2 : Nancy
3 : Good
4 : Wizzie
5 : Jack
```

## 1.7. 函数和array对象

要使用`数组模板类array`，需要包含头文件array，`#include<array>`，而arrat位于命名空间std中。  
array不仅可以存储基本数据类型，还可以存储类对象。

```cpp
void show(std::array<double, 4> da);
void fill(std::array<double, 4> *pa);
```

**示例代码：**

```cpp
// arrobj.cpp 
#include<iostream>
#include<array>
#include<string>
using namespace std;

const int Seasons = 4;
const array<string, Seasons> snames = {"spring", "summer", "fall", "winter"};

void fill(array<double, Seasons> *pa);
void show(array<double, Seasons> da);

void fill(array<double, Seasons> *pa) {
	for (int i = 0; i < Seasons; i++) {
		cout << "Enter " << snames[i] << " expenses: ";
		cin >> (*pa)[i];
	}
}

void show(array<double, Seasons> da) {
	double total = 0.0;
	cout << "\nExpenses\n";
	for (int i =0; i<Seasons; i++) {
		cout << snames[i] << " : $" << da[i] << endl;
		total += da[i];
	}
	cout << "Total Expenses : $" << total <<endl;
}

int main() {
	array<double, Seasons> expenses;
	fill(&expenses);
	show(expenses);
	return 0;
}
```

**执行结果：**
> 因为array模板类是C++11新增，所以编译命令： `g++-5 arrobj.cpp -std=c++11`

```cpp
Enter spring expenses: 212
Enter summer expenses: 256
Enter fall expenses: 208
Enter winter expenses: 244

Expenses
spring : $212
summer : $256
fall : $208
winter : $244
Total Expenses : $920
```

# 2. 递归

> C++函数可以调用自己，和C不同的是，不允许main()调用自己

```cpp
void resurs(argumentlist) {
  statements1
  if (test)
    recurs(arguments)
  statements2
}
```

**示例代码：**

头文件：

```cpp
// recur.h 
void countdown(int n);
```

CPP文件：

```cpp
// recur.cpp 
#include<iostream>
#include "recur.h"
using namespace std;

//void countdown(int n);

int main() {
	countdown(10);
	return 0;
}

void countdown(int n) {
	cout << "Counting down ..." << n <<endl;
	if (n > 0)
		countdown(n-1);
	cout << n << " : Kaboom!\n";
}
```

**执行结果：**

```cpp
Counting down ...10
Counting down ...9
Counting down ...8
Counting down ...7
Counting down ...6
Counting down ...5
Counting down ...4
Counting down ...3
Counting down ...2
Counting down ...1
Counting down ...0
0 : Kaboom!
1 : Kaboom!
2 : Kaboom!
3 : Kaboom!
4 : Kaboom!
5 : Kaboom!
6 : Kaboom!
7 : Kaboom!
8 : Kaboom!
9 : Kaboom!
10 : Kaboom!
```

***

# 3. 函数指针

> 函数也存在地址，函数的地址是存储其机器语言代码的内存的开始地址。例如，可以编写将另一个函数的地址作为参数的函数，这样第一个函数将能够找到第二个函数并运行它。

## 3.1. 释义

函数指针必要工作：
+ 获取函数的地址

只需使用函数名（后面不用跟参数）。比如think()函数的地址是think。要将函数作为参数进行传递，必须传递函数名。`一定要区分传递的是函数的地址还是函数的返回值`

```cpp
process(think);   //传递地址
thought(think()); //传递返回值
```

+ 声明函数指针

声明指向某种数据类型的指针时，必须指定指针指向的类型。同样，声明指向函数的指针时，必须指定指针指向的函数类型。
即声明应该像函数原型那样指出有关函数的信息。

通常要声明指向特定类型的函数的指针，可以首先编写如下的函数原型，然后用`(*pf)`替换函数名。如此，pf就是这个函数的指针。

```cpp
double pam(int);  //函数原型

double (*pf)(int);  //如果(*pf)是函数，pf就是函数指针

double *pf(int);  //返回double指针

pf = pam;  //pf指向pam函数，但是要注意参数类型必须相同
```

+ 使用指针来调用函数

```cpp
doubld pam(int);
double (*pf)(int);

pt = pam;   //pf指向pam()函数

double x = pam(4);
double y = (*pf)(5);  //（1） 等价，调用pam(5)

double y = pf(5);  //（2） 也可以调用函数指针指向的pam函数，但是没有第一种易懂
```

**代码示例：**

```cpp
// fun_ptr.cpp 
#include<iostream>

double betsy(int);
double pam(int);
void estimate(int lines, double (*pf)(int));

using namespace std;

double betsy(int lns) {
	return 0.05 * lns;
}

double pam(int lns) {
	return 0.03 * lns + 0.0004 * lns * lns;
}

void estimate(int lines, double (*pf)(int)) {   //函数指针
	cout << lines << " lines will take";
	cout << (*pf)(lines) << " hours\n";
}

int main() {
	int code;
	cout << "How many lines of code do you need?"<<endl;
	cin >> code;
	cout << "Here is Betsy's estimate:\n";
	estimate(code, betsy);   //入参函数名
	cout << "Here is Pam's estimage:\n";
	estimate(code, pam);
	return 0;
}
```

**执行结果：**

```cpp
How many lines of code do you need?
6
Here is Betsy's estimate:
6 lines will take0.3 hours
Here is Pam's estimage:
6 lines will take0.1944 hours
```

## 3.2. 案例

```cpp
const double * f0(const double ar[], int);
const double * f1(const double [], int);
const double * f2(const double * , int);   //这三种表达方式相同

//声明一个指针指向这三个函数之一，假设指针名是pa，则只需将目标函数原型中的函数名代替为(*pa)
const double * (*pa)(const double *, int);
//可以在声明的同时初始化
const double * (*pa)(const double *, int) = f1;
//简洁的使用auto
auto p2 = f2;  //也是函数指针
```

**示例代码：**

```cpp
// arfupt.cpp 
#include<iostream>
using namespace std;

const double * f1 (const double ar[], int n);
const double * f2 (const double [], int);
const double * f3 (const double *, int);

const double * f1(const double * ar, int n) {
	return ar;
}

const double * f2(const double  ar[], int n) {
        return ar+1;
}

const double * f3(const double ar[], int n) {
        return ar+2;
}

int main() {
	double av[3] = {1.2, 2.3, 3.4};

	const double *(*p1)(const double *, int ) = f1;

	auto p2 = f2;

	cout << "Using pointers to functions:\n";
	cout << "Address Vaule\n";
	cout << (*p1)(av, 3) << " : " << *(*p1)(av, 3) <<endl;
	cout << p2(av, 3) << " : " << *p2(av, 3) <<endl;

	const double *(*pa[3])(const double *, int) = {f1, f2, f3};
	auto pb = pa;
	cout << "\nUsing pointers to functions:\n";
	cout << "Address Value\n";
	for (int i = 0; i<3; i++)
		cout << pa[i](av, 3) << " : " << *pa[i](av,3) <<endl;
	cout << "\nUsing pointers to functions:\n";
        cout << "Address Value\n";
	for (int i = 0; i<3; i++)
                cout << pb[i](av, 3) << " : " << *pb[i](av,3) <<endl;

	auto pc = &pa;
        cout << (*pc)[0](av,3) << " : " << *(*pc)[0](av,3) <<endl;

	return 0;
}
```

**执行结果：**

```cpp
Using pointers to functions:
Address Vaule
0x7ffef6915240 : 1.2
0x7ffef6915248 : 2.3

Using pointers to functions:
Address Value
0x7ffef6915240 : 1.2
0x7ffef6915248 : 2.3
0x7ffef6915250 : 3.4

Using pointers to functions:
Address Value
0x7ffef6915240 : 1.2
0x7ffef6915248 : 2.3
0x7ffef6915250 : 3.4
0x7ffef6915240 : 1.2
```

***

# 4. typedef关键字简化

除了auto，C++11提供了typedef穿件类型别名，`typedef double real;`

```cpp
typedef const double *(*p_fun)(const douvle *, int);   //p_fun现在是一个类型名称（别名）
p_fun p1 = f1;  //p1指向f1()函数

p_fun pa[3] = {f1, f2 ,f3};  //pa是一个指向三个函数的函数指针
p_fun (*pd)[3] = &pa;  //pd指向一个包含三个函数指针的数组
```

# 5. 小结

+ 函数必须提供定义和原型，并且调用该函数
+ 函数原型描述了函数的接口：入参的数目和类型、返回类型
+ 默认情况，C++函数按值传递参数，意味着函数定义中的形参是新的变量，被初始化为函数调用所提供的值。因此通过使用拷贝保护了原始数据的完整性
+ C++将数组名参数视为数组第一个元素的地址，`typename arr[]`和`typename * arr`是等价的
+ C++提供三种表示C风格字符串的方法：字符数组、字符串常量、字符串指针，类型都是`char*`
+ C++提供string类，用于表示字符串，使用`size()`用于判断存储的字符串的长度
+ 处理结构的方式和基本类型完全相同，可以按值传递结构，并将其用作函数返回类型。`如果结构非常大，则传递结构指针，同时函数能够使用原始数据`
+ 支持递归
+ 函数名和函数地址的作用相同。通过函数指针作为参数，可以传递要调用的函数的名称