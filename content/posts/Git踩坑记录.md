---
title: Git踩坑记录
date: 2025-03-09T21:28:21
slug: GitTrap
tags:
  - "#Git"
categories:
  - "#技术探究"
description: 协作开发项目时，使用git遇到的一系列问题和解决方法
summary: 协作开发项目时，拉取到删除文件夹的commit操作，并被重复执行。由于git merge操作不当导致修复困难。代价为dev分支此前的merge记录被清空。
cover:
 image:
draft: false
share: true
---

# 问题概括

在 pull 代码时，拉取到我曾经删除半成品 `frontend` 目录的 commit，由于某些原因，此 commit 重新执行了一次，导致 frontend 目录下某些文件被删除，未及时发现，并通过 PR 提交到远程仓库。
修复的困难：由于使用了 `git merge --ff-only`，仓库和本地分支中含有所有的 commit 记录。使用残缺分支 merge 完成分支操作无效，使用完整分支提交 PR 到 github 的 dev 分支也无效。最后通过 git push --force 实现，清除了 commit 历史记录。
修复导致的代价是 dev 分支此前的 merge 记录被清空。

# 由于拉取代码不当导致的代码丢失问题

我有两个分支：
dev-auth 提交分支
dev-auth-local 本地开发分支

此次提交中，我先采用了 `git merge --ff-only` 操作，将 local 分支的 commit 保留到 dev-auth 分支上。
随后使用 `git pull team dev` （team 是团队仓库的远程名称）准备处理冲突随后提交。
然后我准备将 `dev-auth-local` 开发好的代码 merge 到 `dev-auth` 上
但是在解决冲突后，发现部分文件丢失。

通过重置分支，发现
问题发生在：aa542ab8 这次 commit 上

![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225932.png)
也就是，从本地分支，合并 远程分支的过程中出现了文件的删除。

通过查看，发现是 之前的一次提交，我为了提交最小改动从而在本地删除了 frontend 文件夹，这个操作本身没有问题，但是当他**再次执行一遍**的时候，就造成了当前文件被删除。

**为什么 commit 会再次执行一遍？**
理论上来说，同样的 commit 不会再执行，但是检查哈希值发现，两次 commit 的哈希值并不一致。
![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225933.png)

![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225934.png)

同样，查看协作者的 pr，他的 commit 哈希与我 pull 下来应用的 commit 的哈希也不一致
![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225935.png)
![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225936.png)

看到上一次的 commit 和这一次 commit 并不在同一条分支记录上
![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225932.png)
最左侧蓝色是 dev-auth-local 分支，粉红色是 dev-auth(backup) 分支 ，
第一次 commit：当我切换到 dev-auth 分支时，我进行了删除半成品 frontend 的操作，仅提交必要部分。
第二次 commit：从 dev-auth-lcoal 新建了分支 dev，用来拉取远程仓库的 dev 分支 (错误操作，应当在 dev-auth 上拉取)，冲突解决好后 merge 到 dev-auth 上。但是因为此次 commit 哈希码变化，导致 merge 时对 dev-auth 重新生效，所以导致删除操作又执行一次。

# 由于使用 Fast-forward 的 merge 导致的解决问题困难

Fast-forward 合并是一种合并方式，它可以在合并两个分支时保留提交历史。当我们使用 Fast-forward 合并时，Git 会将源分支的提交直接合并到目标分支上，而不会创建一个新的合并提交。
>需要注意，此操作需要多次保留或修改其中 commit，会造成未知影响，同时，由于 commit 记录保留，再次 merge 操作时会忽略代码
```
git checkout master
git merge --ff-only feature
```

结果：feature 分支记录保持不变，master 分支加入了 feature 的 commit 记录

## 问题
使用 --ff-only 的本意是希望 commit 记录能保留，操作本身没什么问题。
现在的问题是，残缺的代码分别在 dev-auth，remote/dev 分支上，完整代码在 dev-auth-local 上，分别都存在相同的 commit 记录，如何修复残缺代码？

## 解决步骤
1.在 dev-auth 上 merge 来 dev-auth-local，从而 push
由于存在相同的 commit 记录，merge 会显示 alread up to date，哪怕他们的代码不一样。

2.使用 `git reset --hard dev-auth-local` 强制覆盖。
得到完好的 dev-auth 分支

3.在远程仓库回滚上次 PR，使用完好的 dev-auth 分支提交 PR。
失败，由于 commit 记录存在，回滚后代码消失，commit 记录没消失，导致无法识别到全部需要修改的代码。

4.删除所有相关文件，再添加所有相关文件。产生新的 commit 记录。
失败，仍然只能识别到部分需要修改的代码。

5.使用 `git push origin dev-auth:dev --force` 强制覆盖远程 dev 分支。
没办法的办法，丢失了此前三次 merge 记录。
但是同时 commit 记录也清爽了。

## 两分支存在差异，但 merge 提示 alread up to date
![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225938.png)

![](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250309215225939.png)

git merge 操作的本质是应用 commit。因为 merge 只应用 commit 记录更改代码，包含了 commit 就不更改了，不看具体代码一不一致。

## 解决方案
创建一个备份分支
```
git branch backup-dev-auth
```

强制覆盖（谨慎使用）
如果你确定 `dev-auth-local` 的内容应该完全覆盖 `dev-auth`，可以使用以下命令强制覆盖：
`git reset --hard dev-auth-local`

效果为，分支完全变为 dev-auth-local，包括 commit 记录

# 经验教训
1. 使用 git 命令时，你最好知道自己正在做什么。即便你表面上知道自己在做什么，也可能因为概念理解不清晰导致意外错误。(但不能害怕错误)
2. 外加参数使用要谨慎，普遍使用 merge 不加参数是有原因的。
3. 无论如何，通过 commit 保存好自己的代码，恢复只是操作和时间问题。
4. 养成良好的分支管理习惯，避免不必要分支。

# Reference

[git代码分支有不同合并后代码并无更新还存在不同_git merge master没有更新代码-CSDN博客](https://blog.csdn.net/qq_61832991/article/details/128577192)
