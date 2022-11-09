---
layout: single
related: false
title:  ubuntu搭建opengl环境
date:   2019-08-20 22:23:05
categories: linux
tags: linux
toc: true
---

> ubuntu搭建opengl环境

# 1. 安装

```s
sudo apt-get install build-essential
sudo apt-get install libgl1-mesa-dev
sudo apt-get install libglu1-mesa-dev
sudo apt-get install freeglut3-dev
```
<!--more-->
安装完成后，库文件：

```s
wizzie@wizzie:/usr/lib/x86_64-linux-gnu|
⇒  ls -tl lib[gG][lL]*.so
lrwxrwxrwx 1 root root 22 5月  10 20:17 libGLdispatch.so -> libGLdispatch.so.0.0.0
lrwxrwxrwx 1 root root 15 5月  10 20:17 libGLX.so -> libGLX.so.0.0.0
lrwxrwxrwx 1 root root 17 2月  14  2019 libGLESv1_CM.so -> libGLESv1_CM.so.1
lrwxrwxrwx 1 root root 14 2月  14  2019 libGLESv2.so -> libGLESv2.so.2
lrwxrwxrwx 1 root root 10 2月  14  2019 libGL.so -> libGL.so.1
lrwxrwxrwx 1 root root 16 8月  24  2016 libglut.so -> libglut.so.3.9.0
lrwxrwxrwx 1 root root 15 5月  22  2016 libGLU.so -> libGLU.so.1.3.1
```

# 2. 测试

## 2.1. 测试C代码

```c
//test.c
#include <GL/glut.h>


void init(void)
{
    glClearColor(0.0, 0.0, 0.0, 0.0);
    glMatrixMode(GL_PROJECTION);
    glOrtho(-5, 5, -5, 5, 5, 15);
    glMatrixMode(GL_MODELVIEW);
    gluLookAt(0, 0, 10, 0, 0, 0, 0, 1, 0);

    return;
}

void display(void)
{
    glClear(GL_COLOR_BUFFER_BIT);
    glColor3f(1.0, 0, 0);
    glutWireTeapot(3);
    glFlush();

    return;
}

int main(int argc, char *argv[])
{
    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_RGB | GLUT_SINGLE);
    glutInitWindowPosition(0, 0);
    glutInitWindowSize(300, 300);
    glutCreateWindow("OpenGL 3D View");
    init();
    glutDisplayFunc(display);
    glutMainLoop();

    return 0;
}
```

## 2.2. 编译

```shell
$ gcc -o test test.c -lGL -lGLU -lglut
$ ./test
```

编译结束后执行，会出现一个红色的小茶壶，表示配置完成。

## 2.3. 测试C++代码

```cpp
// test1.cpp 
// File Name: example.cpp                                                  
#include <GL/glut.h>
  
void draw()
{
    glClearColor(1, 0, 0, 1);
    glClear(GL_COLOR_BUFFER_BIT);
    glFlush();
}
  
int main(int argc, char** argv)
{
    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_SINGLE | GLUT_RGB);
    glutInitWindowPosition(100, 100);
    glutInitWindowSize(300, 300);
    glutCreateWindow("My First OpenGL Program");
    glutDisplayFunc(draw);
    glutMainLoop();
    return 0;
}
```

## 2.4. 编译

```shell
gcc -o test1 test1.cpp -lGL -lGLU -lglut
./test1
```

编译结束后执行，会出现一个红色的窗口，表示配置完成。