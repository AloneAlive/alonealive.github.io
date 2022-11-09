---
layout: single
related: false
title: C++复合类型之枚举、指针
date:   2019-08-12 22:21:05
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，主要包含C++复合类型之枚举、指针


# 1. 枚举`enum`

> C++的enum工具提供了另一种创建符号常量的方式，可以代替`const`。  
> 它还允许定义新的类型，但是必须严格按照格式。  
> 使用`enum`语法格式和`结构`相似，**例如**  

`enum colorInfo{red, blue, orange};`

+ colorInfo是新类型的名称；colorInfo被成为`枚举`
+ red,blue,orange是符号常量，对应整型树脂`0,1,2`，这些常量叫做`枚举量`  
    + 默认情况下，将整型数值赋给枚举量，从0开始以此类推

1.声明

使用枚举名来声明这种枚举的变量：`colorInfo band;`

2.赋值

只能使用定义枚举量赋值给枚举的变量：

```diff
+ band = red; //将定义枚举中的red值赋给band变量
- band = black;   //在枚举中不存在

double *pn;
char *pa;
double *pc;

double bu = 4.2;
pn = &bu;
pa = new char;
pc = new double[10];
```

3.设置枚举的值

> + 使用赋值运算符显式设置枚举的值：
> > `enum bits {one=1, two=2};`
> >
> + 显式定义部分元素（其他的元素以前一个元素作为参照依次+1）
> > `enum bytes {a, b=100, c}`  **//此时a=0,b=100,c=101**
> >
> + 创建多个值相同的枚举量
> > `enum {zero, null=0, one, numero_uno=1};` **//其中zero和null都是0，后面两个都是1，这是合法的**

4.枚举的取值范围

> **枚举的最大值**的**最小的2的次幂**，将其减去1，得到取值范围的上限。
> > **例如最大值是101，则最小的2的次幂是128，减去1，所以取值的上限是127**
> >
> 如果`枚举量的最小值>=0`，则取值范围的下限是0；否则同上，取2的最小次幂，减一，加上负号。
> > **例如最小值是`-6`，则取最小2次幂减一是7，加上负号，所以取值下限是-7**

***

# 2. **指针**

> 计算机在存储数据的时候必须跟踪的三种基本属性：
> 1. 信息存储的位置
> 2. 存储的值
> 3. 存储的类型
>
> 指针是一个变量，其存储的值是`值的地址`

## 2.1. 地址运算符`&`

地址运算符`&`可以获取一个常规变量的地址，例如`home`是变量，`&home`即是他的地址

```cpp
// address1.cpp 
#include<iostream>

using namespace std;

int main() {
	int dog = 6;
	double food = 4.5;

	cout << "dog = " << dog << ", food = " << food <<endl;

	cout << "dog address = " << &dog << ", food address = " << &food <<endl;

	return 0;
}
```

**打印结果：**

```cpp
dog = 6, food = 4.5
dog address = 0x7fff81decc1c, food address = 0x7fff81decc20
```

**Notes:**

1.常用十六进制描述地址（也存在十进制表示法）。 使用常规变量时，`值是指定的量，而地址是派生量`

2.**面向对象OOP和面向过程的编程区别**在于，OOP强调的是在运行阶段（非编译阶段）进行决策。运行阶段指的是程序正在运行，编译阶段指的是编译器将程序组合起来时。  

    运行阶段决策提供了灵活性，可以根据实时情况进行调整。例如声明数组的时候定义长度。

3.指针用于存储值的地址。指针名表示的是地址，`*`运算符被称为间接值或解除引用运算符，将其应用于指针，可以得到该地址处存储的值。
> 例如：假设manly是一个指针，则`manly`表示的是一个地址，而`*manly`表示存储在该处的值。`*manly`和常规int变量等效。

**例如下列代码：**

```cpp
// pointer.cpp
#include<iostream>
using namespace std;

int main() {
	int updates = 6;   	//变量
	int * p_updates;	//指针
	p_updates = &updates;	//指针存储了变量的地址

	cout << "value updates = " << updates << ", p_updates = " << p_updates << ", &updates = " << &updates <<endl;

	*p_updates = *p_updates + 1;
	cout << "Now updates = "<< updates << ", *p_updates = "<< *p_updates << ", p_updates = " << p_updates <<endl;

	return 0;
}
```

**执行结果：**

```cpp
value updates = 6, p_updates = 0x7ffcf520fbcc, &updates = 0x7ffcf520fbcc
Now updates = 7, *p_updates = 7, p_updates = 0x7ffcf520fbcc
```

## 2.2. 声明和初始化

> 指针声明必须指定指针指向的数据的类型  
> > `int * p_updates;`   
> > `*p_updates`的类型是`int`，由于`*`被用于指针，因此`p_updates`变量本身必须是指针，即`p_updates`指向int类型，或者是指向int的指针，或`int*`
> > `p_updates`是指针（地址）；`*p_updates`是int，而不是指针。

**Notes:**
> (1) `*`两边的空格是可选的。  
> `int *ptr;` 强调`*ptr`是int类型的变量。  
> `int* ptr;` 强调`int*`是一种指向int的指针。（在C++中，`int*是复合类型，是指向int的指针`）
> >
> (2) 在C++中创建指针时，计算机将分配内存用来存储地址的内存，但是不会分配用来存储指针所指向的数据的内存。例如：
> > `long *fellow;`  
> > `*fellow = 2333; //ERROR`
> 此处没有给`2333`赋地址。

## 2.3. 指针和数字

> 整数可以加减乘除，而指针表示地址，描述的是位置，将两个地址相乘没有意义。

```cpp
//ERROR 不可以直接赋值，编译器会有错误消息：通用类型不匹配
int* ptr;
ptr = 0xB8000000;

//Right 转换
int* ptr;
ptr = (int*) 0xB8000000;
```

## 2.4. 使用`new`分配内存

> 在C++中，除了可以使用C语言的方法`malloc()`函数分配内存，还可以使用`new`运算符  
> 例如：  
> (1) `int *ptr = new int;`  
> `new int`告诉程序需要合适存储int的内存。`new`运算符根据类型确定需要多少字节的内存，然后找到这样的内存并返回地址。接着将地址赋给`ptr`，`ptr`是被声明为指向int的指针。
>
> (2) `int hig;`  
> `int *pn = &hig;`  
> 这种方法是使用`hig`名称来访问`int`
> 
> 为一个数据对象（可以是结构，或者基本类型等）获得并指定分配内存的通用格式：  
> **`typename * poniter_name = new typename;`**

**代码示例：**

```cpp
// use_new.cpp 
#include<iostream>

using namespace std;

int main(){
	int nights = 1001;
	int *pt = new int;  //分配内存
	*pt = 1001;  //赋值

	cout << "nights = " << nights << ", its' address = " << &nights <<endl;
	cout << "*pt = " << *pt << ", its' address = " << pt <<endl;

	double *pd = new double;
	*pd = 10000001.0;

	cout << "*pd = " << *pd << ", address of pointer *pd = " << &pd <<endl;
	cout << "size of pt = " << sizeof(pt);
	cout << " : size of *pt = " << sizeof(*pt) <<endl;
	cout << "size of pd = " << sizeof(pd);
	cout << " : size of *pd = " << sizeof(*pd) <<endl;

	return 0;
}
```

**执行结果：**

```cpp
nights = 1001, its' address = 0x7fff3426f484
*pt = 1001, its' address = 0x150bc20
*pd = 1e+07, address of pointer *pd = 0x7fff3426f488
size of pt = 8 : size of *pt = 4
size of pd = 8 : size of *pd = 8
```

**Note:**

变量`nights`和`pd`的值都存储在`栈stack`的内存区域；而`new`从`堆heap`或者`自由存储区free store`的内存区域分配内存。

## 2.5. 使用`delete`释放内存

> 使用delete时，后面要加上指向内存块的指针（这些内存块最初是由new分配的）

```cpp
int *ps = new int;
...
delete ps;
```

这将会释放ps指向的内存，但是不会删除指针ps本身。例如，可以将ps重新指向另一个新分配的内存块。

**一定要配对的使用new和delete**，否则会发生`内存泄漏 memory leak`。

**只能用delete来释放使用new分配的内存（对空指针使用delete是安全的）**

**使用delete的关键是，将他用于new分配的内存，这并不意味者要使用用于new的指针，而是用于new的地址。**

**例如：**

```cpp
int *ps = new int;
delete ps;  //ok
delete ps;  //not ok

int jugs = 6;
int *pi = &jugs; //ok
delete pi;   //not allowed

int *pk = new int;	//分配内存
int *pq = pk;   //声明第二个指针
delete pq; //delete第二个指针，并不会影响pk的值（但是pk的值必须要new分配内存）
```

## 2.6. 使用`new`创建动态数组

例如下面的语句创建指针，它指向包含十个int值的内存块中的第一个元素：
`int *psome = new int[10];`

不能修改数组名的值，但是可以修改指针变量。

将指针变量加一后，增加的量等于它指向类型的字节数（例如int数组一个变量四个字节）

**代码：**

```cpp
//arraytnew.cpp 
#include<iostream>

int main() {
	using namespace std;
	double *p3 = new double[3];
	p3[0] = 0.2;
	p3[1] = 0.5;
	p3[2] = 0.8;

	cout << "p3[1] = "<< p3[1] <<endl;

	p3+=1;

	cout << "Now p3[0] = "<< p3[0] <<endl;
        cout << "Now p3[1] = "<< p3[1] <<endl;

	p3-=1;

	delete []p3;  //free memory
	return 0;
}

```

**结果：**

```cpp
p3[1] = 0.5
Now p3[0] = 0.5
Now p3[1] = 0.8     //+1之后的数组第一个元素就变成了第二个元素
```

## 2.7. **指针小结**

+ 对数组使用`sizeof`得到的是数组的长度（int4个字节乘以元素数量，单位是字节），而对指针使用`sizeof`得到的是指针的长度，即指针指向的是一个数组（元素数量）。

```cpp
int *pt = new int[10];
*pt = 5;  //pt[0]=5 or pt=5
pt[0] = 6;  //reset

pt[9] = 44;
int coats[10];
*(coats + 4) = 12;   //coats[4] = 12
```
+ 数组名被解释为其第一个元素的地址`arrayname`，而对数组名应用地址运算符`&`时，即`&arrayname`，得到的是整个数组的地址。
例如：

```cpp
	short tell[10];
	tell //display &tell[0]，一个2字节内存块的地址
	&tell //display address of whole array，一个20字节内存块的地址
```

+ 对指针接触引用意味着获取指针指向的值，例如`*pn`，另一种接触引用的方法是使用数组表示法，例如`pn[0]`。
+ 区别指针和指针指向的值，`int *pt = new int; *pt = 5;`其中pt是指向int的指针，而*pt是完全等同于一个int类型的变量。
+ 数组的`静态联编`是使用数组声明来创建数组时，数组的长度在编译时设置`int tacos[10];`
+ 数组的`动态联编`是使用`new[]`创建数组，在运行时为数组分配空间，其长度在运行时者之。使用完这种数组后，应该使用`delete[]`释放占用的内存。

```cpp
// ptrstr.cpp 
#include<iostream>
#include<cstring>

using namespace std;

int main() {
	char animal[20] = "bear";
	const char *bird = "wren";
	char *ps;

	cout << animal << " and " << bird <<endl;
	//cout << ps <<endl;  //may crash, may garbage, may skip, print nothing

	ps = animal;
	cout << ps << endl;
	cout << (int *)animal << " and " << (int *)ps <<endl;

	ps = new char(strlen(animal) + 1);  //get new storage返回字符串的长度
	strcpy(ps, animal);  //copy string to new storage将字符串从一个位置复制到另一个位置
	cout << "After using strcpy, animal " << animal << " at "<< (int*)animal <<endl;
	cout << ps << " at " << (int*)ps <<endl;

	delete []ps;

	return 0;
}
```

**执行结果：**

```cpp
bear and wren
bear
0x7ffde23d00b0 and 0x7ffde23d00b0
After using strcpy, animal bear at 0x7ffde23d00b0
bear at 0x1b2b030
```