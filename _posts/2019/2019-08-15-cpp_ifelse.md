---
layout: single
related: false
title:  C++分支语句、逻辑表达式、字符函数库、switch、文本I/O
date:   2019-08-15 22:23:05
categories: cpp
tags: cpp
toc: true
---

> C++基础语法，包含C++分支语句、逻辑表达式、字符函数库、switch、文本I/O

# 1. if语句

> 两种格式： `if`和`if else`

```cpp
// if.cpp 
#include<iostream>

using namespace std;

int main() {
	char ch;
	int spaces = 0;
	int total = 0;
	cin.get(ch);
	while (ch != '.') {
		if (ch == ' ')
			++spaces;
		++total;
		cin.get(ch);
	}

	cout << spaces << " spaces, total is " << total <<endl;
	return 0;
}
```

**执行结果：**

```shell
a
b
c
 //space
 //space
.
2 spaces, total is 10   //总数为10是因为包含换行符


或者如下结果：
dwdadw   .
3 spaces, total is 9
```

## 1.1. 嵌套ifelse

```cpp
if (ch == 'A')
  a++;
else
  if (ch =='B')
    b++;
  else if (ch == 'C')
    c++;
  else
    x++;
```

# 2. 逻辑表达式

> 三种：逻辑or`||`， 逻辑and`&&`，逻辑NOT`!`

例如： 

```shell
5 > 3 || 5 > 10   //如果左侧为true，就不会判断右侧
等同于
(2 < 3) || (5 > 10) //说明逻辑运算符优先级低于关系运算符

5 > 8 && 5 < 10  //优先判断左侧，如果左侧为false，就不会判断右侧

!(x > 5)   //取反
```

## 2.1. 使用保留字表达

逻辑运算符|另一种表达方式
:-:|:-:
&&|and
或|or
!|not

# 3. 字符函数库cctype

> 使用`isalpha()`来检查字符是否为字母字符  
> 使用`isdigit()`来测试字符是否是数字字符  
> 使用`isspace()`来测试字符是否是空白（如换行符、空格、制表符）  
> 使用`ispunct()`来测试字符是否是标点符号

函数名|入参|返回值
:-:|:-:|:-:
isalnum() | 字母或数字 | true
isalpha() | 字母 | true
iscntrl() | 控制字符  | true
isdigit() | 数字（0～9）| true
isgraph() | 除空格外的打印字符 | true
islower() | 小写字符 | true
isprint() | 打印字符，包含空格| true
ispunct() | 标点符号 | true
isspace() | 标准空白字符，如空格、换行、回车、水平制表符、垂直制表符| true
isupper() | 大写字母 | true
isxdigit()| 十六进制数字，即0～9、a~f、A～F | 返回true
tolower() | 大写字符 | 返回其小写，否则返回参数
toupper() | 小写字符 | 返回大写，否则返回参数

# 4. 三目条件运算符（?:）

`5 > 3 ? 10 : 12`，如果true，则返回10，false返回12

# 5. switch

> switch中的每个case标签必须是一个单独的值。这个值必须是整数（含char）。因此switch无法处理浮点测试。另外case标签必须是常量。  
> `break`和`continue`都呢该构跳过代码。不同之处前者跳出整个循环，后者跳出本次循环。

```cpp
// switchtest.cpp 
#include<iostream>

using namespace std;

int main() {
        char choice;
        cin >> choice;

        while (choice != 'Q' && choice != 'q') {
                switch(choice) {
                        case 'a':
                        case 'A':
                                cout << "result is a/A\n";
                                break;
			case 'b':
                        case 'B':
                                cout << "result is b/B\n";
                                break;
                        case 'd':
                        case 'D':
                                cout << "result is d/D\n";
                                break;
                        case 'c':
                        case 'C':
                                cout << "result is c/C\n";
                                break;
			default:
				cout << "Not abcd" <<endl;
				break;
                }
		cin >> choice;
        }
	return 0;
}
```

**执行结果：**

```shell
a
result is a/A
B
result is b/B
c
result is c/C
D 
result is d/D
F
Not abcd
q
```

# 6. 文件输入输出`I/O`

## 6.1. 写入

```cpp
char ch;
std::cin >> ch;
std::cout << "Result is " << ch <<std::endl;

char word[50];
cin >> word;  //不断读取，直到遇到空白字符
cin.getline(word, 50);  //不断读取，直到遇到换行符
```

## 6.2. 写入到文本文件

```cpp
//  outfile.cpp
#include<iostream>
#include<fstream>

using namespace std;

int main() {
	char automobile[50];
	int year;
	double a_price;
	double b_price;

	ofstream outFile;
	outFile.open("carinfo.txt");

	cout << "Enter the make and model of automobile: \n";
	cin.getline(automobile, 50);

	cout << "Enter the model year: \n";
	cin >> year;

	cout << "Enter the original asking price: \n";
	cin >> a_price;
	b_price = 0.913 * a_price;

	outFile << fixed;
	outFile.precision(2);
	outFile.setf(ios_base::showpoint);
	outFile << "Make and model: " << automobile <<endl;
	outFile << "Year : " << year <<endl;
	outFile << "Was asking $" << a_price <<endl;
	outFile << "Now asking $" << b_price <<endl;

	outFile.close();
	return 0;
}
```

**执行结束后生成的文件：**

```shell
// carinfo.txt 
Make and model: Flitz Perky
Year : 2009
Was asking $13500.00
Now asking $12325.50
```

## 6.3. 读取文本

**读取文件：**

```cpp
// readfile_test.txt 
12 31.2 321
23 23.21 31
23 31
```

**代码：**

```cpp
// readfile.cpp 
#include<iostream>
#include<fstream>
#include<cstdlib>

const int SIZE = 60;

int main() {
	using namespace std;
	char filename[SIZE];
	ifstream inFile;

	cout << "Enter file name: \n";
	cin.getline(filename, SIZE);
	inFile.open(filename);

	if (!inFile.is_open()) {
		cout << "Could not open file " << filename <<endl;
		exit(EXIT_FAILURE);
	}

        double value;
	double sum = 0.0;
	int count = 0;  //读取的数量

	inFile >> value;  //获取第一个alue
	while (inFile.good()) {
		++count;         //读取数量+1
		sum += value;    //极端总和
		inFile >> value; //获取下一个value	
	}

	if (inFile.eof()) 
		cout << "End of file reached.\n";
	else if (inFile.fail())
		cout << "Input terminated by data mismatch.\n";
	else
		cout << "Input terminated for unknown reason.\n";

	if (count == 0)
		cout << "No data processed.\n";
	else
	{
		cout << "Item read : " << count <<endl;
		cout << "Sum : " << sum <<endl;
	}

	inFile.close();
	return 0;
}
```

**执行结果：**

```cpp
Enter file name: 
readfile_test.txt
End of file reached.
Item read : 8
Sum : 495.41
```