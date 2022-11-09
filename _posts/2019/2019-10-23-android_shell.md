---
layout: single
related: false
title:  Android Shell脚本应用
date:   2019-10-23 23:52:00
categories: android
tags: shell
toc: true
---

> Android shell脚本的常用语法介绍

# 1. 基本结构.sh（系统中文.ch.sh）

`#!/bin/bash` 脚本开头

`#!/bin/csh` 是 C shell 的命令解

***

# 2. export

> 功能说明：设置或显示环境变量。  
> 语　　法：export [-fnp][变量名称]=[变量设置值]  
> 补充说明：在 shell 中执行程序时，shell 会提供一组环境变量。export 可新增，修改或删除环境变量，供后续执行的程序使用。export 的效力仅限于该次登陆操作。

1. 用户登录到Linux系统后，系统将启动一个用户shell。在这个shell中，可以使用shell命令或声明变量，也可以创建并运行 shell 脚本程序。
2. 运行shell脚本程序时，系统将创建一个子shell。此时，系统中将有两个shell，一个是登录时系统启动的 shell，另一个是系统为运行脚本程序创建的shell。
3. 当一个脚本程序运行完毕，脚本shell将终止，返回到执行该脚本之前的shell。从这种意义上来说，用户可以有许多shell，每个shell都是由某个 shell（称为父 shell）派生的。
4. 在子shell中定义的变量只在该子shell内有效。
5. 如果在一个shell脚本程序中定义了一个变量，当该脚本程序运行时，这个定义的变量只是该脚本程序内的一个局部变量，其他的 shell 不能引用它，要使某个变量的值可以在其他 shell 中被改变，可以使用 export 命令对已定义的变量进行输出。
6. export命令将使系统在创建每一个新的shell时，定义这个变量的一个拷贝。这个过程称之为变量输出。

***

# 3. 用语句给变量赋值，如将`/etc`下目录的文件名循环出来

```shell
for file in `ls /etc`
或
for file in $(ls /et)
```

使用`$`符号来取一个变量的值，常见的就是`$VAR`

变量定义：`app_name=”test”`

# 4. `$、$0、$1、$2..`

`$0`就是你写的shell脚本本身的名字，`$1`是你给你写的shell脚本传的第一个参数，`$2`是你给你写的shell脚本传的第二个参数。

# 5. `ps -ef |grep surfaceFlinger |awk ‘{print $2}’`的含义

+ `$2`：表示第二个字段
+ `print $2`：打印第二个字段
+ `-e`: 显示所有进程。
+ `-f`:全格式
+ `awk '{print $2}' $fileName`: 一行一行的读取指定的文件， 以空格作为分隔符，打印每行的第二个字段（即 pid）。
+ `ps`: 打印的信息：
+ 字段含义如下：

```shell
UID PID PPID C STIME TTY TIME CMD
zzw 14124 13991 0 00:38 pts/0 00:00:00 grep --color=auto dae
```

# 6. 两种方式声明函数

1.“function”不可以省略(建议)

```shell
 function find { ...
 }
```

2.不得添加参数

```shell
 find() { ...
 }
```

两种声明方式效果等价。
**注意：**  
（1）函数名和”{“之间必须有空格；  
（2）不得声明形式参数；  
（3）必须在调用前声明；  
（4）无法重载；  
（5）后来的声明会覆盖之前的声明

***

# 7. while 语句

`while`循环用于不断执行一系列命令，也用于从输入文件中读取数据；命令通常为测试条件。其格式为：

```shell
while condition
do
    command
done
```

以下是一个基本的while循环，测试条件是：如果int小于等于5，那么条件返回真。int从0开始，每次循环处理时，`int`加1运行上述脚本，返回数字1到5，然后终止。

```shell
#!/bin/bash
int=1
while(( $int<=5 ))
do
    echo $int
    let "int++"
done
```

***

# 8. `$android_serial=$(adb shell getprop “re.serialno”)`获取设备序列号

# 9. `adb shell input swipe 359 1600 359 340 500`

`input`后可以跟很多参数，text相当于输入内容，keyevent相当于手机物理或是屏幕按键，tap相当于touch事件，`swipe`相当于滑动。
`input/swipe`模拟的是滑动事件，`input swipe <x1> <y1> <x2> <y2> [duration(ms)] (Default: touchscreen)`，需要将起始的坐标传进去。
+ 向左滑动：
`shell@lentk6735_66t_l1:/ $ input swipe 600 800 300 800`
+ 向右滑动：
`shell@lentk6735_66t_l1:/ $ input swipe 300 800 600 800`
+ 滑动：
` adb shell input swipe 100 100 200 200 300 //从 100 100 经历 300 毫秒滑动到 200 200`
+ 长按：
`adb shell input swipe 100 100 100 100 1000 //在 100 100 位置长按 1000 毫秒`

***

# 10. `if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi（适合中断，写成一行）`

```shell
if condition
then
 command1 
 command2
 ...
 commandN
else
 command
fi
```

+ `-eq`:等于
+ `-ne`:不等于
+ `-le`:小于等于
+ `-ge`:大于等于
+ `-lt`:小于
+ `-gt`：大于
+ `-a`: 双方都成立（and） 逻辑表达式 –a 逻辑表达式
+ `-o`: 单方成立（or） 逻辑表达式 –o 逻辑表达式

***

# 11. `sed -i ‘s/^.\{15\}//g’ tmp.txt`

`adb shell wm size > tmp.txt` //分辨率导出文件

`sed -i ‘s/^.\{15\}//g’ tmp.txt` //此处 `\{`是转义成 `{

`wm_size=$(cat tmp.txt)`

***

## 11.1. `sed -i `就是直接对文本文件进行操作

`sed -i 's/原字符串/新字符串/' /home/1.txt`

`sed -i 's/原字符串/新字符串/g' /home/1.txt`

## 11.2. 正则

+ `^`: 匹配输入字符串的开始位置。如果设置了`RegExp`对象的`Multiline`属性，`^`也匹配`\n`或`\r`之后的位置。
+ `.`: 匹配除“\n”之外的任何单个字符。要匹配包括“\n”在内的任何字符，请使用像“(.|\n)”的模式。
+ `\`: 将下一个字符标记为一个特殊字符、或一个原义字符、或一个向后引用、或一个八进制转义符。例如，`n`匹配字符`n`。`\n`匹配一个换行符。串行`\\`匹配`\`而`\(`则匹配`(`。
+ `{n}`:n是一个非负整数。匹配确定的n次。例如，`o{2}`不能匹配`Bob`中的`o`，但是能匹配`food`中的两个`o`。

## 11.3. sed

+ `-n`：只打印模式匹配的行
+ `-e`：直接在命令行模式上进行 sed 动作编辑，此为默认选项
+ `-f`：将 sed 的动作写在一个文件内，用–f filename 执行 filename 内的 sed 动作
+ `-r`：支持扩展表达式（配合正则表达式）
+ `-i`：直接修改文件内容

**例如：**
+ 删除该行的第一个字符：`sed -r 's/^.//g' <<< $line`
+ 删除文件每行的第二个字符: `sed -r 's/^(.)(.)/\2/g' passwd`
+ 删除文件每行的倒数第二个字符: `sed -r 's/(.)(.)$/\2/g' passwd`
+ 交换每行的第一个字符和第二个字符:`sed -r 's/^(.)(.)/\2\1/g' passwd`
+ 交换每行的第一个单词和最后一个单词:`sed -r 's/^([a-Z0-9]+)([^a-Z0-9]+)(.+)([^a-Z0-9]+)([a-Z0-9]+)/\5\2\3\4\1/g' passwd`

# 12. 多选择语句`case`

```shell
echo '你输入的数字为:'
read aNum
case $aNum in
    1) echo '你选择了 1'
    ;;
    2) echo '你选择了 2'
    ;;
    3) echo '你选择了 3'
    ;;
    4) echo '你选择了 4'
    ;;
    *) echo '你没有输入 1 到 4 之间的数字'
    ;;
esac
```

***

# 13. `adb shell pidof ”mediaserver“`

按名称检查正在运行的进程，可以使用`pidof`命令：`adb shell pidof com.android.phone`

如果找到此类进程，则返回`PID`,否则返回空字符串。

***

# 14. `adb shell am start -n`

`adb shell am start -n com.android.camera...` //使用组件名方式启动照相机功能

可以使用`adb shell activity|grep ACTIVITY` 或者`dump SF`获取Activity名称

***

# 15. `adb shell input tap x y`

模拟点击事件（点击屏幕），可以用来进入清除任务。

***

# 16. `adb shell dumpsys window windows |grep Current |tee tmp.txt`

**实现功能：**

+ 获得当前活动窗口的信息，包名以及活动窗体。

+ 过滤`Current`信息，获得`window`的`dump`信息。

+ `|tee`的作用：输出到控制台

***

# 17. `date + %Y%m%d_%H%M%S`输出日期

输入格式：20181128_174323

# 18. `date + %N | cut -c -3`

`%N`: 纳秒

`cut`： 从指定的范围中提取字节（-b）、或字符（-c）、或字段（-f）

+ `cut -b -3 `//-3 表示从第一个字节到第三个字节
+ `cut -b 3- `//3-表示从第三个字节到行尾
+ `cut -b -3,3-` //输出整行，不会出现连续两个重叠的

***

# 19. `cat **systrace.html |grep postComp |wc -l`

 `（linux 命令）WC`：`wc - print newline, word, and byte counts for each file`

参数及含义：
+ `-c`: print the byte counts 统计字节数
+ `-l`: print the newline counts：统计行数
+ `-m`: print the character counts：将每个文件的字符数及文件名输出到屏幕上，如果当前系统不支持多字节字符其将显示与-c 参数相同的结果
+ `-w`: print the word counts：统计字数

***

# 20. （视频播放时）旋转屏幕

`adb shell settings put system user_rotation 1`  //获取参数是get

# 21. shell判断数组中是否包含某个元素

```shell
ary=(1 2 3)
a=2
if [[ "${ary[@]}" =~ "$a" ]] ; then
 echo "a in ary"
else
 echo "a not in ary"
fi
```

***

# 22. 解决播放结束后，判断当前activity然后重新播放

+ `adb shell dumpsys activity top | grep "ACTIVITY"` //获得当前的 activity
+ `adb shell am start -n com.home...` //返回桌面 activity

***

# 23. 包含关系

```shell
testPlay=$(adb shell dumpsys activity top | grep "ACTIVITY")testPlaySuccess="com...Activity"
                if [[ $testPlay =~ $testPlaySuccess ]];then
                        echo "包含"
                else
                        echo "不包含"
                fi
```

***

# 24. 查找手机文件

`adb shell find /storage -name "*.mp4" | grep FileName`

# 25. 播放视频

`adb shell am start -a android.intent.action.VIEW -d "file:///storage/...../Test.mp4" -t "video/"`

# 26. 逐行读某个文件

```shell
while read line
do
…
done < file // `<`是读，`>`是写入，`>>`是写入到某个文件的末尾
```
