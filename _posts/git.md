---
date: 2024-07-01

date: 2024-07-01

layout: post
title: "git"
date: 2021-07-01T11:00:00.000Z
last_modified_at: 2024-07-01T20:28:59.000Z
categories:
  - 技术 
tags:
---

### 时光机
```shell
git reflog #或 git log --pretty=oneline
git reset HEAD@${index}
```

### 修改最近一次提交
```shell
git add . # 或者指定文件
git commit --amend --no-edit
```
不要在公共分支做该操作

### 修改最近一次commit信息
```shell
git commit --amend
```

### 提交错分支
#### 方法一
```shell
# 撤回提交但保留提交内容 
git reset HEjAD~ --soft
git stash

#切到正确分支
git checkout some-branch
git stash pop
```

#### 方法二
```shell
git checkout some-branch
#抓取master分支上最新的commit
git cherry-pick master
git checkout master
git reset HEAD~ --hard
```

### 撤回很早以前的一个commit
```shell
git log --pretty=oneline
#找到那个commit的hash值
git revert [hash]
```

### 撤回某一个文件的改动 
```shell
git log --pretty=oneline
#记下文件改动前那次提交的hash值
git checkout [hash] -- path/to/file
# 改动前的文件会保存到你的暂存区
git commit -m "这样就不用复制粘贴了" 
```

### 快速合并commit分支
```shell
git log --pretty=oneline
git rebase -i [hash] #hash为要合并的最后一个分支的上一个分支
git checkout --ours && git add . && git rebase --continue # 应用最新分支的修改
```