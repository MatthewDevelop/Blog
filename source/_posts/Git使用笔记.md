---
title: Git使用笔记
date: 2019-01-21 11:33:54
tags:
    - Git
---

> 记录Git使用过程中的一些操作指令



# 标签
- 创建标签  
`git tag <Tag Name>` 默认在最后的提交上打Tag        
`git tag <Tag Name> <SHA-1 Code>` 指定在某一笔提交上打Tag
<!-- more -->
- 创建带注释的Tag
`git tag -a <Tag Name> -m <Tag Message> <SHA-1 Code>`
- 查看Tag
`git tag`
- 查看Tag详细信息
`git show <Tag Name>`
- 推送Tag到远程
`git push origin <Tag Name>`
- 推送本地的所有Tag
`git push origin --tags`
- 删除本地标签
`git tag -d <Tag Name>`
- 删除远程标签
先删除本地标签，然后推送到远程
`git push origin :refs/tags/<Tag Name>`

# 分支
- 查看所有分支
`git branch -a`
- 删除远程分支
`git push origin --delete BranchName`
- 删除本地分支
`git branch -d BranchName`

# 本地仓库关联到远程仓库
1. 首先拿到远程仓库的地址
    > 以GitHub为例：`git@github.com:[username]/[repository name]`
2. 将本地代码转换为Git仓库  
    > `git init`
3. 将本地仓库关联到远程仓库
    > `git remote add origin [remote address]`
4. 同步远程仓库,和远程分支关联
    > `git pull origin master`
5. 提交代码
    > `git add ` 和 `git commit`
6. 推送到远程仓库
    > `git push origin master`