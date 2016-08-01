---
layout: post
title: Git删除文件
categories: [Github]
description: How to rm files in repository with github.
keywords: git rm, git add -A
---

使用Git删除版本库中的指定文件，也要显示的使用git命令删除，下面举个例子说明。
首先新建一个文件：

```sh
>>>>>>> 2be974c0a840746264283c162c17e3d3b113d93f
touch to_be_deleted.txt
git add to_be_deleted.txt
git commit -m "commit to_be_deleted.txt"
$ git commit -m "commit to_be_deleted.txt"
[master b32ee44] commit to_be_deleted.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 to_be_deleted.txt

git push -u origin master
```

正常删除操作如下：

```sh
git rm to_be_deleted.txt
rm 'to_be_deleted.txt'
git status -s
D  to_be_deleted.txt
$ git commit -m "delete to_be_deleted.txt"
[master 7568bac] delete to_be_deleted.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 to_be_deleted.txt

git push -u origin master
```

如果没有使用git rm，而是直接rm，情况会怎么样呢?

```sh
rm to_be_deleted.txt
$ git status -s
 D to_be_deleted.txt
```

git status -s 虽然可以看出工作区和版本库中代码不一致，to_be_deleted.txt被删除了。
但是D是红色的，这次的删除操作并没有被提交到缓冲区，即没有stage。需要手动git add：

```sh
git add to_be_deleted.txt
git status -s
 D to_be_deleted.txt
```
手动git add 以后，git status -s 查看D为绿色。

如果直接使用系统命令rm的文件比较多，又分布在不同的文件中，逐个git add会特别费时费力。此时只需使用以下操作：

```sh
rm a/a.txt
rm b/b.txt
git rm c/c.txt
git status -s
 D a/a.txt
 D b/b.txt
D  c/c.txt
git add -A
git status -s
D  a/a.txt
D  b/b.txt
D  c/c.txt
```
使用git add -A以后，使用rm命令直接删除的a.txt和b.txt也stage完毕。
