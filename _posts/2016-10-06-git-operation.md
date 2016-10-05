---
date: 2016-10-06
title: Git操作指令详解
categories: 操作系统
tags: [git, Linux]
excerpt: "Linux在1991年创建了开源的Linux系统,如今已发展成了应用最广泛的开源系统. 当然,Linux系统的壮大离不开全世界热心的志愿者的贡献, 这么多人把代码发送给Linux, Linux如何合并代码呢?"
---

全面的总结一下git指令以及常用的运用场景

## 分布式版本控制系统Git历史和简述

Linux在1991年创建了开源的Linux系统, 如今已发展成了应用最广泛的开源系统. 当然, Linux系统的壮大离不开全世界热心的志愿者的贡献, 这么多人把代码发送给Linux, Linux如何合并代码呢?

一开始肯定是手动合并, 后来转向了一个商业的版本控制系统BitKeeper, 但是在2005年, Linux社区牛人试图破解BitKeeper协议被BitMover公司发现了, 终止了对Linux社区的免费使用权

然后这位大牛Linux花了两周时间用C语言写了一个分布式管理系统Git, 在一个月之内,Linux系统代码全部由Git管理. 嗯, 两周的时间.

> Git是分布式的版本控制系统, 而与之对应早出现的CVS和SVN都是集中式的版本控制系统, 这种集中控制类似CS架构(服务端-客户端), 需要联网才能工作, 提交慢, 一旦服务器宕机, 整个都瘫痪.

> 而Git作为分布式的版本控制系统, 涉及思路类似p2p协议, 每一个用户都是服务器, 拥有完整的资源, 各个用户互为对等方, 只需要把修改推送给对方即可, 提交快, 稳定性高.

**由于Git的涉及理念就是开源, 所以在权限控制上几乎不要求, 而SVN这些集中版本控制系统却能够很(bian)好(tai)地管理权限, 目的是应用在那些大公司里,对程序员只给自己相关部分代码的读写权限.**


## Git的三个重要概念:

![](/assets/img/posts/2016-10-05-git-operation/1.jpeg)

* 工作区(working directory),就是你所能看到的电脑文件夹

* 暂存区(stage)

* 分支(branch)

## 创建版本库

初始化一个Git仓库，使用`git init`命令。

添加文件到Git仓库，分两步：

1. 第一步，使用命令`git add <file>`,把单个file文件修改添加到暂存区,注意,可以通过 git add * 把所有修改的文件一次性全部添加;

2. 第二步，使用命令`git commit -m "describe your changes here"`，把暂存区的所有内容提交到当前分支。

要随时掌握工作区的状态，使用`git status`命令。

如果`git status`告诉你有文件被修改过，用`git diff`可以查看修改内容。


## 版本回退:

当前分支用`HEAD`指针来区分,主分支一般的名字叫master,当`HEAD`指向master分支时,master就为当前分支,

HEAD指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令`git reset --hard commit_id`。

* 穿梭前，用`git log`可以查看提交历史，以便确定要回退到哪个版本。

* 要重返未来，用`git reflog`查看命令历史，以便确定要回到未来的哪个版本。


## 撤销修改

1. 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- <file>`。

2. 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD file`，就回到了场景1，第二步按场景1操作。

3. 场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退一节，不过前提是没有推送到远程库


## 远程仓库(Github)

要关联一个远程库，使用命令`git remote add <远程仓库名> <此仓库在github上的URL>`

> 一般自己的github上的远程仓库名叫做origin,而别人的(多半是被fork的远程仓库名)叫upstream

> 可以使用 `git remote` 查看当前关联的远程库, `git remote -v` 能查看更多详细信息

关联后，使用命令`git push -u <远程仓库名> <分支名> 推送本地此<分支名>`的所有内容至远程仓库对应的<分支名>里,

* 要克隆一个仓库，首先必须知道仓库的地址，然后使用 `git clone <仓库在github上的URL>` 命令克隆。

## 分支管理:

分支在实际中有什么用呢？假设你准备开发一个新功能，但是需要两周才能完成，第一周你写了50%的代码，如果立刻提交，由于代码还没写完，不完整的代码库会导致别人不能干活了。如果等代码全部写完再一次提交，又存在丢失每天进度的巨大风险。

现在有了分支，就不用怕了。你创建了一个属于你自己的分支，别人看不到，还继续在原来的分支上正常工作，而你在自己的分支上干活，想提交就提交，直到开发完毕后，再一次性合并到原来的分支上，这样，既安全，又不影响别人工作。

* 查看分支：`git branch`, 可增加参数 `-a` 和 `-v`

* 创建分支：`git branch <新的分支名>`

* 切换分支：`git checkout <已有分支名>`

* 创建+切换分支(本地)：`git checkout -b <新的分支名>`

* 创建+切换分支(从远程仓库) `git checkout -b <新的本地分支名> <远程仓库名>/<远程分支名>`

* 合并某分支到当前分支：`git merge <已有分支名>`

* 删除分支：`git branch -d <已有分支名>`

* 强行删除分支(在未merge之前)：`git branch -D <已有分支名>`

## 分支合并

### 1.本地合并分支

合并当前分支和其他分支: `git merge <其他分支名>`, 若两个分支之间存在冲突,必须手动解决冲突后再提交:

1.使用`git status`查看冲突的文件,Git用`<<<<<<<，=======，>>>>>>>`标记出不同分支的内容,例如:

```
$ cat readme.txt
Git is a distributed version control system.
Git is free software distributed under the GPL.
Git has a mutable index called stage.
Git tracks changes of files.
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```

2.我们保留其中一个分支内容(feature1),删掉Git给我们的标记符号,如下:

```
$ cat readme.txt
Git is a distributed version control system.
Git is free software distributed under the GPL.
Git has a mutable index called stage.
Git tracks changes of files.
Creating a new branch is quick AND simple.
```

3.再次提交add和commit提交, 用`git log --graph`命令可以看到分支合并图。

### 2.远程合并分支

```
git fetch <远程仓库名>
git merge <远程仓库名>/<远程分支名>
```

执行上面两步,先必须通过`git remote add <远程仓库名> <此仓库在github上的URL>`与远程仓库关联,然后再fetch和merge,解决冲突与本地一样

NOTE: **`git pull <远程仓库名> <远程分支名>:<本地分支名>` 等同于git fetch 和 git merge两条指令顺序执行.**

> 通过设置追踪关系: `git branch --set-upstream <本地分支名> <远程仓库名>/<远程分支名>`, `git pull` 不加参数就可以能够使用


## 分支策略(dev分支)

在实际开发中，我们应该按照几个基本原则进行分支管理：

首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，再把dev分支合并到master上，在master分支发布1.0版本；

你和你的小伙伴们每个人都在dev分支上干活，每个人都有自己的分支，时不时地往dev分支上合并就可以了。

所以，团队合作的分支看起来就像这样：

![](/assets/img/posts/2016-10-05-git-operation/2.png)

## bug分支和feature分支

软件开发中，bug就像家常便饭一样。有了bug就需要修复，在Git中，由于分支是如此的强大，所以，每个bug都可以通过一个新的临时分支来修复，修复后，合并分支，然后将临时分支删除。

当手头工作没有完成时，先把工作现场git stash一下，然后去修复bug，修复后，再git stash pop，回到工作现场。

添加一个新功能时，你肯定不希望因为一些实验性质的代码，把主分支搞乱了，所以，每添加一个新功能，最好新建一个feature分支，在上面开发，完成后，合并，最后，删除该feature分支。


## 多人协作

多人协作的工作模式通常是这样：

1. 首先,fork人家的项目

2. `git clone <自己github账号下fork项目的url>` 到本地

3. 修改代码,并先后使用 `git add *` 和 `git commit -m "I modify this and that"` 提交到自己的master(默认), 更高级的可以自己建branch

4. 然后用`git push origin master`推送自己的修改到自己远程仓库名origin的master分支,

5. 最后在github页面上提交新的pull request, 等待对方(被fork项目)同意.

当对方(被fork项目)发生了修改并提交时,

1. 通过`git remote add <远程仓库名(一般为upstream)> <对方远程分支>` 将本地分支与对方远程分支关联

2. `git pull` 将远程分支的修改拉到本地合并, 若有冲突,则解决冲突，并在本地提交.

3. 最后用`git push <自己远程仓库名(一般为origin)> <分支名>` 推送至自己的远程仓库,则现在你的远程仓库与对方的远程仓库就是同步的.


<hr>

参考[廖雪峰官方日志](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

参考[阮一峰的网络日志](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)

