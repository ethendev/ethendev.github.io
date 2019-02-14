---
layout: post
title: Git 内部原理
tags:  [Git]
categories: [Git]
keywords: Git,原理
---

Git 是 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的版本控制软件。可以有效、高速地处理从很小到非常大的项目版本管理。从根本上来讲 Git 是一个内容寻址（content-addressable）文件系统，并在此之上提供了一个版本控制系统的用户界面。 本篇文件将深入介绍 Git 内部的工作机制。





### 目录结构

当在一个新目录或已有目录执行 git init 时，Git 会创建一个 .git 目录。 这个目录包含了几乎所有 Git 存储和操作的对象。 如若想备份或复制一个版本库，只需把这个目录拷贝至另一处即可。 本章探讨的所有内容，均位于这个目录内。 该目录的结构如下所示：
```
$ ls -Fl
total 7
-rw-r--r-- 1 admin 197121 130 2月  23 10:32 config
-rw-r--r-- 1 admin 197121  73 2月  23 10:32 description
-rw-r--r-- 1 admin 197121  23 2月  23 10:32 HEAD
drwxr-xr-x 1 admin 197121   0 2月  23 10:32 hooks/
drwxr-xr-x 1 admin 197121   0 2月  23 10:32 info/
drwxr-xr-x 1 admin 197121   0 2月  23 10:32 objects/
drwxr-xr-x 1 admin 197121   0 2月  23 10:32 refs/
```

该目录下可能还会包含其他文件，不过对于一个全新的 git init 版本库，这将是你看到的默认结构。 
`description` 文件仅供 GitWeb 程序使用，我们无需关心。 `config` 文件包含项目特有的配置选项。 `info` 目录包含一个全局性排除（global exclude）文件，用以放置那些不希望被记录在 .gitignore 文件中的忽略模式（ignored patterns）。 `hooks` 目录包含客户端或服务端的钩子脚本（hook scripts）。

剩下的四个条目是 Git 的核心组成部分: 
* `objects` 目录存储所有数据内容
* `refs` 目录存储指向数据（分支）的提交对象的指针
* `HEAD` 文件指示目前被检出的分支
* `index` 文件保存暂存区信息（因为是新目录，所以上面没有）

