---
title: Git 学习笔记
tags:
  - git
  - 版本控制
  - 工具
status: maintained
updated: 2026-10-06
---

# Git 学习笔记

## 基本的Linux命令

1. `cd` 改变目录
2. `cd ..` 回退到上一个目录；直接输入 `cd` 通常返回当前用户主目录
3. `pwd` 显示当前文件目录路径
4. `clear` 清屏
5. `ls` 列出现在目录里的所有文件
	1. 绿色为程序
	2. 蓝色为目录
	3. 白色为文件
6. `touch` 创建文件
7. `rm` 删除一个文件
8. `mkdir` 创建一个文件夹
9. `rm -r` 删除一个文件夹
10. `mv` 移动一个文件，格式为 mv A B 表示将A移入B目录下
11. `reset` 重新初始化终端，和clear差不多
12. `history` 查看历史命令，除非清理
13. `help` 查看命令的帮助信息 格式为 help+命令
14. `exit` 退出
15. `#` 表示注释
## Git配置

查看配置 `git config -l`
配置文件都保存在本地
使用 `git config user.name` 名字 和 `git config user.email` 邮箱 来配置文件
## Git基本理论

Git本地有三个工作区域
1. 分为工作区(workspace)、暂存区(staging area)、本地仓库(local repository)
2. 远程只有远程仓库(remote repository)
- workspace：工作区 平时存放项目代码的地方
- staging area：暂存区 临时存放你的改动
- local repository：本地仓库 安全存放数据的位置，有你提交到的所有版本数据
- remote repository：远程仓库，托管代码的服务器 GitHub
## Git工作流程

1. 在工作区中添加修改文件
2. 将要进行版本控制的文件放入暂存区 `git add`
3. 将暂存区的文件提交到本地仓库 `git commit`
## Git仓库
### 本地git仓库
Git 使用`git init`命令来初始化一个 Git 仓库。在执行`git init`完成命令后，Git 仓库会生成一个 .git 目录，该目录包含了资源的所有元数据。
```cmd
# 在当前目录新建一个代码仓库
$ git init
```
### 远程克隆仓库
将远程服务器上的代码仓库复制一份到本地
```cmd
#克隆一个项目和它的整个代码周期
$ git clone <URL>
```
## Git文件操作
### 文件的四种状态
- Untracked 未跟踪，此文件在目录中但git不管它，用`git add`将状态变为`staged`
- Unmodified 文件已经入库，但是没有修改，工作区文件与暂存区内容一致。修改后，文件会变为 `modified`；执行 `git rm` 会删除文件并把删除操作加入暂存区。
	- Modified 文件已经修改，可以用 `git add` 进入 `staged` 暂存状态；如果只想撤销工作区修改，可以使用 `git restore <文件名>`。
- Staged 暂存状态，执行 `git commit` 将修改提交到本地仓库
### 查看文件状态
```
#查看指定文件状态
git status [filename]
#查看所有文件状态
git status

# 添加所有文件到暂存区
git add .
#提交暂存区的文件到本地仓库
git commit -m "提交说明"
```
## 使用SSH密钥连接GitHub

1. 使用命令行或Gitbash在[C盘/用户](C:\Users\EvenHawk)这个目录下创建一个文件夹，名字为`.ssh`。
2. 输入`ssh-keygen -t rsa`表示创建一个密钥用RSA来加密。
3. 找到你文件夹下的id_rsa.pub,打开将内容复制下来
4. 在GitHub上找到添加.ssh密钥并添加
## 编辑器集成 Git

使用 VS Code 绑定 Git 就行
创建文件然后在左边可以直接commit
commit 需要填写信息，使用 `git commit -m "这里填信息"`
使用终端来进行push
## Git分支

Git 分支管理是 Git 强大功能之一，能够让多个开发人员并行工作，开发新功能、修复 bug 或进行实验，而不会影响主代码库。
几乎每一种版本控制系统都以某种形式支持分支，一个分支代表一条独立的开发线。
使用分支意味着你可以从开发主线上分离开来，然后在不影响主线的同时继续工作。
  ![587](https://static.jyshare.com/images/svg/git-brance.svg)
  ```
  # 检查所有分支
  git branch
  
  # 创建一个新分支
  git branch [branch]
  
  # 创建一个新分支,并切换到该分支
  git switch -c [branch]
   
  # 切换到指定分支
  git checkout [branch]
  
  # 将指定分支合并到当前分支
  git merge [branch]
  
  # 删除已合并的指定分支（不能删除当前所在分支）
  git branch -d [branch]
  
  #删除远程分支
  git push origin --delete [branch]
  git branch -dr origin/[branch]
  ```
