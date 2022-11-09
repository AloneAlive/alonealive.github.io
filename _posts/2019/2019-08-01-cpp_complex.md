---
layout: single
related: false
title: C++复合类型之数组、字符串、结构、共用体
date:   2019-08-01 21:41:05
categories: cpp
tags: cpp
toc: true
---

> 序： C++提供了基于整型和浮点类型创建的复合类型。影响最深远的复合类型是`类`。然而，C++还支持几种普通的复合类型。  
> 例如，__数组__ 可以存储多个同类型的值。  
> __结构__ 可以存储多个不同类型的值。  
> __指针__ 则是一种将数据所处位置告诉计算机的类型。

<!--more-->

# 1. 数组array

1.数组声明应指出以下三点：

+ 存储每个元素中的值的类型
+ 数组名
+ 数组的元素数量

2.格式：

`short months[12];`

`typename arrayname[size];`

__size__ 必须是整型常数或const值，也可以是常量表达式。他不能是变量。

3.C++11新增初始化的功能

+ 初始化数组可以省略`等于号=`，例如`double ear {1.2e2, 1.6e3, 2.3e4,3.5e10};`
+ 可以不在大括号内包含任何东西，意味着所有元素是`0`
+ **列表初始化禁止缩窄转换**

```cpp
        long plifs[] = {23,12,3.0}; //浮点数转换成长整型是缩窄操作，不能编译通过
        char slifs[] = {'g',1123456,'\o'};  //1123456超过char变量的取值范围，不能编译通过
        char tlifs[] = {'h',112,'o'};  //OK
```

4.数组的替代品 -- 模板类vector

在C++标准模板库STL提供了**vector**，以及C++11增加了模板类`array`。

***

# 2. 字符串

> C++处理字符串的方式有两种，一种来自C语言，常称为`C-风格字符串`，另一种基于`string`类库的方法。

## 2.1. `C-风格字符串(字符数组)`

1.**C-风格字符串**以`空字符 \0 ` 结尾，其ASCII码是`0`。

`char dog[3] = {'d','o','g'};  //not a string`
    
`char dog[3] = {'d','o','g', '\0'};  //is a string`

2.**字符串常量或者称字符串字面值**

> 使用双引号表示字符串常量，而字符是单引号

`char bird[11] = "Mr. cheeps"; //the \0 is hideen,隐式包含结尾的空字符`

`char fish[] = "buddles";  //left the complier count`

> `sizeof()`指出整个数组长度，而`strlen`返回的是存储在数组中字符串的长度，而不是数组本省的长度。  
> `strlen()`只计算可见的字符，不计算空字符`\0`在内。(只针对`char数组`，并且需要加入头文件 `#include<cstring>`)

3.**读取一行字符串输入**

> istream中的类（例如`cin`）提供了一些面向行的成员函数：`getline()和get()`  
> 这两个函数都读取一行输入，直到到达换行符  
> 随后，`getline`将丢弃换行符，`get()`将换行符保留在输入序列。

（1）`cin.getline(name,20);` 将一行数据读入到一个包含20个元素的name数组。如果这行包含的字符不超过19个（`\0`）

```cpp
// instr2.cpp
    #include<iostream>
    int main() {
	using namespace std;
	const int size = 20;
	char name[size];
	char group[size];

	cout << "Enter your name : \n";
	cin.getline(name, size);  //read info inline
	cout << "Enter group: \n";
	cin.getline(group, size);

	cout << "name is "<< name <<endl;
	cout << "groupf is " << group <<endl;
	return 0;
    }
```

**结果：**

```shell
    Enter your name : 
    abc
    Enter group: 
    def
    name is abc
    groupf is def
```

（2）`cin.get(name,size)`不会换行，而是将换行符读入到下一行开始。可以通过以下两种方式换行：  

```cpp
        //first
        cin.get(name,size);
        cin.get();
        cin.get(group,size);
        //second
        cin.get(name.size).get();
```

(3) **读取空行**

> 当get()或者getline()读取空行后将设置`失效位 failbit`。这意味接下来的输入将被阻断。  
> 恢复输入方法： `cin.clear();`

(4) 输入字符串比分配的空间（数组size）长，则getline()和get()将把余下的字符留在输入队列中，`getline()`还会设置`失效位`，并关闭后续的输入。

## 2.2. `string`类

> string类需要头文件`string`。并且string位于命名空间`std`中。

**代码示例：**

```cpp
// strtype1.cpp 
#include<iostream>
#include<string>

using namespace std;

int main(){
	string str1;
	string str2 = "cat";

	cout << "Enter string data:\n";
	cin >> str1;

	cout << "str1=" << str1 <<", str2 = "<< str2 << ", str2[2] = "<< str2[2] <<endl;
	return 0;
}

```

**结果：**

```cpp
Enter string data:
dog
str1=dog, str2 = cat, str2[2] = t
```

**Note:**

类设计让程序能够自动处理`string`的大小。例如str1声明的时候长度为0，读取到输入后长度是3。

***

1.**C++11新增的字符串初始化：**

```cpp
char char1[] = {"hello boy"};
char char2[] {"hello girl"};

string str1 = {"good boy"};
string str2 {"good girl"};
```

2.**字符串拼接合并**

```cpp
string str3;
str3 = str1 + str2;
str1 += str2;
```

3.**`cstring头文件`的字符数组`char[]`复制和附加操作**

```cpp
strcpy(char1, char2);  //将char2数组赋值到char1
strcat(char1, char2);  //将char2数组附加到char1末尾
```

> 对比来说，字符串string的拼接和附加操作更加简单。

## 2.3. 两种字符串书写方式的`I/O`和`字符串长度`

**代码：**

```cpp
//strtype2.cpp 
#include<iostream>
#include<string>
#include<cstring>

using namespace std;

int main() {
	char ch[10];
	string str;

	cout << "length of ch[10] = " << strlen(ch) <<endl;
	cout << "length of str = " << str.size() <<endl;

	cout << "Enter a line for ch[10]: \n";
	cin.getline(ch,10);

	cout << "Enter a line for str:\n";
	getline(cin, str); //这个getline()不是类方法，他将cin作为参数，指出到哪里去查找输入

	cout << "Now length of ch[10] = " << strlen(ch) <<endl;
        cout << "Now length of str = " << str.size() <<endl;

	return 0;
}
```

**结果：**

```cpp
length of ch[10] = 0
length of str = 0
Enter a line for ch[10]:
cat dog
Enter a line for str:
cat dog2
Now length of ch[10] = 7
Now length of str = 8
```

## 2.4. `wchar_t, char16_t, char32_t`的初始化

```cpp
wchar_t title[] = L"Paper"; //L
char16_t name[] = u"Nancy"; //u
char32_t subject[] = U"math"; //U
```

## 2.5. C++11新增的原始字符串`raw`，以`R`为前缀

`cout << R"("king" \n and queue)"`

结果：

`"king" \n and queue`

> 原始字符串使用`"(`和`)"`作为限定符，换行符`\n`也只打印两个单独的符号。  
> 或者使用`"+*(`和`)*+"`作为限定符

***

# 3. 结构

> **结构可以存储多种类型的数据。**
>> 结构是用户定义的类型  
>> 而结构声明定义了这种类型的数据属性  
>> 定义了类型之后，便可以创建这种类型的变量  
>>
> **创建结构包含两步：**
> > 1. 定义结构描述（它描述并且标记能够存储在结构中的各种数据的类型）
> > 2. 按照描述创建变量（结构数据变量）

1.**例如以下结构描述**

```cpp
struct inflatable   //结构的关键字`struct`，标识符`inflatable`是这种数据格式的名称
{
    char name[20];
    float volume;
    double price;
};
```

2.**创建这种类型的变量：**

```cpp
inflatable hat;   //允许省略关键字`struct`，因为结构声明定义了一种新的数据格式
inflatable mainframe;
```

因为`hat`是数据结构类型，所以允许使用`hat.name`来访问成员。

3.**示例代码：**

```cpp
// strucetype1.cpp 
#include<iostream>

struct inflatable   //结构的关键字`struct`，标识符`inflatable`是这种数据格式的名称
{
    char name[20];
    float volume;
    double price;
};

int main(){
	using namespace std;
	inflatable hat = {
		"pen",
		1.88,
		29.99
	};

	inflatable pal = {
		"pencil",
		3.13,
		32.99
	};

	cout << "List name : " << hat.name << " ,and " << pal.name <<endl;
	cout << "List volume: " << hat.volume << ",and "  << pal.volume <<endl;
	cout << "Total price : " << hat.price + pal.price <<endl;

	return 0;
}
```

**执行结果：**

```cpp
List name : pen ,and pencil
List volume: 1.88,and 3.13
Total price : 62.98
```

## 3.1. 结构声明的位置

> 结构声明的位置很重要，可以将声明放在main函数中，紧跟在开始括号的后面  
> 也可以选择将声明放在main函数前面  
> > 位于函数外面的声明被成为`外部声明`，如果类包含两个或更多的函数，外部声明可以被后面的函数使用  
> > 而内部声明只能当前函数使用

## 3.2. C++11的初始化

> 同字符串和数组，结构也支持`列表初始化`，即在初始化的时候`=等于号`是可选的。

    `inflatable duck {"Dada", 0.12, 9.98};`

## 3.3. 同时定义结构和和创建变量

```cpp
struct perks {
        int keynum;
        char car[10];
} mr_smith,ms_jones;   //定义的两个结构体变量

//或者同时进行初始化
struct perks {
        int keynum;
        char car[10];
} mr_smith {
    7,
    "Peak"
};

//声明一次性没有名称的结构，此时直接使用`position.x`进行访问
struct    //没有结构体名称
{
    int x,
    int y
} position;
```

## 3.4. 结构数组

> 可以创建结构数组，例如`inflatable gifts[100];`，此时访问成员元素使用`gifts[0].name`  
> 此时的`gifts`不是结构，而是数组  
> ‘gifts[0]’是结构

## 3.5. 结构中的位字段

> C++允许指定占用特定位数的结构成员。这使得创建与某个硬件的寄存器对应的数据结构非常方便。  
> 字段的类型应为整型或者枚举，接下来是冒号，冒号后面是数字，它指定了使用的位数  
> 每个成员都被称为`位字段`
> **位字段常用于低级编程中**

```cpp
struct torgle {
    unsigned int SN : 4;  //4位给SN变量
    unsigned int : 4;   //不指定的4位
    bool googin : 1;   //非法输入
}
```

# 4. 共用体union

> union能够存储不同类型的数据，但是只能同时存储其中的一种类型。  

**例如：**

```cpp
union one4all {
    int val;
    long val2;
    double val3;
}
```

**可以使用`one4all`存储不同的类型，存储int，long，或者double**

```cpp
one4all pail;
pail.val = 10;
cout << pail.val; //10

pail.val3 = 1.35;
cout << pail.val3; //1.35
```

> 因此，pail有时候是int类型，有时候也可以使long，double  
> 成员名称标识了变量的容量  
> **共同体每次只能存储一个值，因此必须有足够的空间来存储里面最大的成员，所以共同体的长度为其最大成员变量的长度**  
> <font color=red>共同体的用途之一是：</font>
> > 当数据项使用两种或者更多的格式，但是不会同时使用时，可以节省时间。

**例如一些商品的ID是整数，而另一些的ID是字符，则可以定义：**

```cpp
struce thing {
    char name[20];

    union id {
        long id_num;  //整型的ID
        char id_char[20];  //字符型的ID
    } id_val;    //id_val是声明在结构体中，可以使用`结构体.id_val.id_num`初始化

};
```

## 4.1. 匿名共用体

> 匿名共用体没有名称，其成员将成为位于相同地址的变量。  
> 每次只有一个成员是当前的成员。

```cpp
struce thing {
    char name[20];

    int type;
    union {      //匿名
        long id_num;  //整型的ID
        char id_char[20];  //字符型的ID
    };

};

......
thing prize;
if (type == 1) {
    cin >> prize.id_num;
} else {
    cin >> prize.id_char;   //此时直接调用匿名共用体的成员
}
```

因为是匿名共用体，所以`id_num`和`id_char`被视为结构体`thing`的成员，他们的地址相同（相对于结构体来理解），因而不需要中间标识符`id_val`（即共用体的变量声明）。

**共用体常用于节省内存（例如操作系统数据结构或硬件数据结构）**
