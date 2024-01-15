---
title: revert代码引出的问题
date: 2019-02-25 11:46:23
tags: 
- Git
categories:
- Git
---

## 场景
将已合并主干的代码进行revert后，再次反悔想合并到主干时该怎么操作？

![](images/git/revert.png)

master分支

1  2  3  4  5  6 ...

1 是master分支

2 表示修改后的代码合并到master分支

3 master分支回滚到和提交1代码一致

4 5 6 表示后续合并到master分支的提交记录

实战场景：想要将2分支修改的代码合并到master分支该如何操作？

## 思路

## 分析
- 在分支2的基础上直接merge master直接会前进到6，1 2之间变更的文件就都丢掉了。
- 可以考虑将将1到2之间的提交变更为一次新的commit，然后与6做diff。

## 解决方案：

方案1：愚公移山，将变更2的代码一个个拷贝出来，然后重新覆盖

方案2：git rebase -i 1，将1到2之间的提交变更为一次新的commit，然后与6做diff

## 额外问题
Q: git revert commit 时出现 error: commit XXXX is a merge but no -m option was given

A: 因为revert的commit是一个merge commit，它有两个parent，Git不知道base是选哪个parent，就没法diff。
所以要显式告诉Git用哪一个parent。 
则 git revert -n commit -m 1 这样就选parent 1，那么parent 1又是哪一个呢？ 
一般来说，如果你在master上merge branch_xxx,那么parent 1就是master，parent 2就是branch_xxx