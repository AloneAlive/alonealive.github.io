---
layout: single
related: false
title: git stash储藏命令
date:   2019-07-22 11:12:33
categories: android
tags: debug
toc: true
---

> 储藏（stashing）可以获取工作目录的中间状态，即被修改过的被追踪的文件和暂存的变更。并将它保存在一个未完结变更的堆栈中，随时可以重新应用。


# 1. 储藏暂时未完成的工作

> 如果你想在当前目录切换分支或者暂停正在进行的工作，而去先做另一件事。  
> 你就需要先储藏这些变更。  
> 为了向堆栈推送一个新的储藏，只需要执行：__git stash__

`git status` 可以查看到干净的工作目录；

`git stash list` 可以查看储藏的列表;  

`git stash show stash@{0}` 可以查看某个储藏的修改信息

> 如果工作目录不干净，包含已修改、未提交的文件，此时进行apply会给出归并冲突。

`git stash apply stash@{0}` 可以请求某个储藏（如果不指定某个，会默认最近的储藏）

> 对文件的变更被重新应用，但是被暂存的文件没有重新被暂存。需要告诉命令重新应用被暂存（commit）的变更。

`git stash apply --index` 告诉命令重新应用被暂存的变更

`git stash drop stash@{0}` 移除某个储藏

# 2. 取消储藏

> 如果已经apply某个储藏，但是在修改一些代码后需要取消这个储藏，此时使用:  
> __git stash show -p stash@{0} | git apply -R__  
> 可以达到取消该储藏的补丁效果。

# 3. `git stash branch <name>`

> 这条命令会根据最近的stash创建一个新的分支，然后删除最近的stash  
> 可以指定某个stash， 在后面加上 `stash@{0}`  
