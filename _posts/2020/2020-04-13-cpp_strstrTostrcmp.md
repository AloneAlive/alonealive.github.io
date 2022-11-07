---
layout: single
related: false
title:  C++字符串比较函数strcmp和strstr的用法
date:   2020-04-14 23:32:00
categories: cpp
tags: cpp
toc: true
---

> 今天碰到一个细节bug，使用`strcmp`来比较两个字符串是否完全相同。但是忽略了一个问题，如果存在一个字符串包含在另一个字符串呢？此时就会发现需要用`strstr`函数。

<!--more-->

# 1. strcmp函数比较字符串大小

设这两个字符串为str1、str2，

+ 若str1 == str2，则返回零；
+ 若str1 < str2，则返回负数；
+ 若str1 > str2，则返回正数。

测试类：
```c++ testFunc_strcmp.cpp 
#include<iostream>
#include <cstring>

#define PROPERTY_VALUE_MAX 92

int main() {
        using namespace std;

        char char1[PROPERTY_VALUE_MAX] = {0};
        char char2[PROPERTY_VALUE_MAX] = {0};

        bool con = true;
        char isCon[20] = "y";

	while (con) {
        cout << "Compare two char by func strcmp(char1, char2):" <<endl;
        cout << "Enter char1: ";
        cin.getline(char1, PROPERTY_VALUE_MAX);

        cout << "Enter char2: ";
        cin.getline(char2, PROPERTY_VALUE_MAX);

        cout << "Result: strcmp(" << char1 << ", " << char2 << "): " <<endl;
        cout << strcmp(char1, char2) <<endl;

        cout << "Continue? y/n" <<endl;
        cin.getline(isCon, 20);
        if(strcmp(isCon, "n") == 0) {
           con = false;
        }
        }
        return 0;
}

```

运行：
```s
Compare two char by func strcmp(char1, char2):
Enter char1: sun
Enter char2: wen
Result: strcmp(sun, wen): 
-4
Continue? y/n
y
Compare two char by func strcmp(char1, char2):
Enter char1: wen
Enter char2: gan
Result: strcmp(wen, gan): 
16
Continue? y/n
y
Compare two char by func strcmp(char1, char2):
Enter char1: wizzie
Enter char2: wizzie
Result: strcmp(wizzie, wizzie): 
0
Continue? y/n
y
Compare two char by func strcmp(char1, char2):
Enter char1: wizzie_test
Enter char2: wizzie
Result: strcmp(wizzie_test, wizzie): 
95
Continue? y/n
n
```

# 2. strstr函数比较字符串是否相同或者存在包含关系

如果两个字符串可能存在相同，并且可能会有包含关系，则需要使用`strstr`函数来比较字符串。如果不包含（或相同），则返回NULL。

测试类：
```c++ testFunc_strstr.cpp
#include<iostream>
#include <cstring>

#define PROPERTY_VALUE_MAX 92

int main() {
        using namespace std;

        char char1[PROPERTY_VALUE_MAX] = {0};
        char char2[PROPERTY_VALUE_MAX] = {0};

        bool con = true;
        char isCon[20] = "y";

	cout << "*** Determine whethe char2 is in char1. ***" << endl;

	while (con) {
        cout << "Input two char by func strstr(char1, char2):" <<endl;
        cout << "Enter char1: ";
        cin.getline(char1, PROPERTY_VALUE_MAX);

        cout << "Enter char2: ";
        cin.getline(char2, PROPERTY_VALUE_MAX);

        cout << "Result: strstr(" << char1 << ", " << char2 << "): " <<endl;
        if (strstr(char1, char2) != NULL)
            cout << "Success: char2 is in char1!" << endl;
        else
            cout << "Error: char2 is not in char1!" << endl;

        cout << "Continue? y/n" <<endl;
        cin.getline(isCon, 20);
        if(strcmp(isCon, "n") == 0) {
           con = false;
        }
        }
        return 0;
}
```

运行：
```s
*** Determine whethe char2 is in char1. ***
Input two char by func strstr(char1, char2):
Enter char1: sun
Enter char2: sun
Result: strstr(sun, sun): 
Success: char2 is in char1!
Continue? y/n
y
Input two char by func strstr(char1, char2):
Enter char1: sunwengang
Enter char2: wen
Result: strstr(sunwengang, wen): 
Success: char2 is in char1!
Continue? y/n
y
Input two char by func strstr(char1, char2):
Enter char1: sun
Enter char2: sunwengang
Result: strstr(sun, sunwengang): 
Error: char2 is not in char1!
```