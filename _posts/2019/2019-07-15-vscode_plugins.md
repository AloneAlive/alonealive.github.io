---
layout: single
related: false
title:  VS code常用插件
date:   2019-07-15 22:04:14
categories: tools
tags: vscode
toc: true
---

> vscode的一些常用插件和配置方法

# 1. vscode离线插件下载

进入vscode插件官网（https://marketplace.visualstudio.com/）

# 2. markdown相关插件

## 2.1. markdown PDF

> 可以转换md文件成pdf, html, png, jpg文件

## 2.2. markdownlint

> 可以提示markdown语法规范

## 2.3. markdown toc

> 可以生成md文件的目录

## 2.4. markdown preview enhanced

> 实时预览，但是在vscode自带预览markdown文件效果，必要性不大。

***

# 3. UML建模插件

## 3.1. plantUML

> 使用一种便捷的设计语言plantUML， 参考http://plantuml.com/
>> 基本使用方法参考：https://www.jianshu.com/p/30f6a9c06083

# 4. vscode插件markmap绘制思维导图

使用 Markmap 绘制思维导图，只需三个符号：

1. #：标题
2. -：列表
3. ---：分隔符

参考：

```shell
## android car(包含了与车相关的基本API)

- [Car.java](/packages/services/Car/car-lib/src/android/car/Car.java)

## cluster(仪表盘相关API)

- links
- **inline** ~~text~~ *styles*
- multiline
  text
- `inline code`
-
    ```js
    console.log('code block');
    ```
- Katex - $x = {-b \pm \sqrt{b^2-4ac} \over 2a}$
```

***


# 5. 思维导图工具

1.Try markmap在线制作（使用markdown）：https://markmap.js.org/repl/
2.百度脑图在线制作：https://naotu.baidu.com/（可以导入markdown导出png图片）

**vscode插件：**
1.mindmap（文件以km结尾）（参考https://blog.csdn.net/liuxiao723846/article/details/107414365）
2.Markmap（同Try markmap使用markdown，文件以mm.md结尾）

# 6. 代码注释插件

+ [VScode插件自动添加注释](https://blog.csdn.net/a314753967/article/details/125422370)

安装koroFileHeader插件

设置选项搜索cursorMode修改如下：

```s
 // 函数注释
    "fileheader.cursorMode": {
        // 默认字段
        "description":"",
        "param":"",
        "return":""
```

***

# 7. vscode配置技巧

## 7.1. vscode添加分隔线

在file->preferences->settings的settings.json中添加：
`"editor.rulers": [4, 100]`

## 7.2. vscode删除缩进多行tab

按：shift + tab

***

## 7.3. VSCode阅读Android内核源码方法

> 参考：[使用Visual Studio Code阅读Android内核源码](https://www.jianshu.com/p/af723ff252e6)

## 7.4. Visual Studio Code（VSCode）关闭右侧预览功能

关闭方法：点击文件-首选项-设置,搜索"editor.minimap.enabled",默认值为打钩,我们只需要把钩去掉即可

## 7.5. vscode始终打开新标签

workbench.editor.enablePreview改为false

## 7.6. vscode标签多行显示

wrap tabs

### 7.6.1. vscode自动换行

ALT+Z 