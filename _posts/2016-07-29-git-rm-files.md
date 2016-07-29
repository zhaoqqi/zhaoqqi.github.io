---
layout: post
title: Git删除文件
categories: [Github]
description: How to rm files in repository with github.
keywords: git rm, git add -A
---

使用Git删除版本库中的指定文件，也要显示的使用git命令删除，下面举个例子说明。
首先新建一个文件：
```
touch to_be_deleted.txt
git add to_be_deleted.txt
git commit -m "commit to_be_deleted.txt"
$ git commit -m "commit to_be_deleted.txt"
[master b32ee44] commit to_be_deleted.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 to_be_deleted.txt

git push -u origin master
```
