---
title: 配置：Vim-Git-SVN
date: 2016-09-21 10:57:19
categories: 笔记
tags:
    - 开发环境配置
    - bash
    - Vim
    - Git
    - SVN
---
# bash
`vim /etc/profile`，添加如下
```
# base
alias ll="ls -la"
```

# Vim
`vim /etc/profile`，添加`export TERM=xterm`。

## 样式
`vim ~/.vimrc`，配置如下
```
syntax on
set tabstop=4
set softtabstop=4
set shiftwidth=4
set autoindent
set cindent
set cinoptions={0,1s,t0,n-2,p2s,(03s,=.5s,>1s,=1s,:1s

if &term=="xterm"
set t_Co=8
set t_Sb=^[[4%dm
set t_Sf=^[[3%dm
endif
```
<!-- more -->

## 命令
[vim命令大全](http://www.cnblogs.com/softwaretesting/archive/2011/07/12/2104435.html)

# Git
## alias
```
vim ~/.gitconfig
```
```
[alias]
    co = checkout
    ad = add 
    aa = add -A
    au = add -u
    ci = commit
    ca = commit -a
    st = status
    pl = pull
    ps = push
    dt = difftool
    l = log --stat
    cp = cherry-pick
    b = branch
```

p.s.
```
# 三种add命令的区别
1.  git add -A   保存所有的修改
2.  git add .     保存新的添加和修改，但是不包括删除
3.  git add -u   保存修改和删除，但是不包括新建文件

# 查看命令帮助
git xxx -h
```

## 添加中文支持
bash中输入`git config --global core.quotepath false`（已使用UTF-8字符集）。

## ignore
`vim ~/.gitignore_global`
添加
```
*~
.DS_Store
# Xcode
#
# gitignore contributors: remember to update Global/Xcode.gitignore, Objective-C.gitignore & Swift.gitignore

## Build generated
#build/
DerivedData/

## Various settings
*.pbxuser
!default.pbxuser
*.mode1v3
!default.mode1v3
*.mode2v3
!default.mode2v3
*.perspectivev3
!default.perspectivev3
xcuserdata/
*.xcuserdatad

## Other
*.moved-aside
*.xcuserstate

## Obj-C/Swift specific
*.hmap
*.ipa
*.dSYM.zip
*.dSYM

# CocoaPods
#
# We recommend against adding the Pods directory to your .gitignore. However
# you should judge for yourself, the pros and cons are mentioned at:
# https://guides.cocoapods.org/using/using-cocoapods.html#should-i-check-the-pods-directory-into-source-control
#
# Pods/

# Carthage
#
# Add this line if you want to avoid checking in source code from Carthage dependencies.
# Carthage/Checkouts

Carthage/Build

# fastlane
#
# It is recommended to not store the screenshots in the git repo. Instead, use fastlane to re-generate the 
# screenshots whenever they are needed.
# For more information about the recommended setup visit:
# https://github.com/fastlane/fastlane/blob/master/fastlane/docs/Gitignore.md

fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots
fastlane/test_output

# Code Injection
#
# After new code Injection tools there's a generated folder /iOSInjectionProject
# https://github.com/johnno1962/injectionforxcode

iOSInjectionProject/
```

Gerrit审查模式配置
切换到相应的git库目录下，执行`git config remote.origin.push refs/heads/*:refs/for/*`命令，此后git push操作会默认推送到remote库中的refs/for/xxx审查分支。

p.s.
删除误提交的xcuserdata
```
// -n：加上这个参数，执行命令时，是不会删除任何文件，而是展示此命令要删除的文件列表预览。
git rm -r -n --cached  */src/\*

// 删除文件的版本控制
git rm -r --cached  */src/\*      
```

# SVN
`vim /etc/profile`，添加`export SVN_EDITOR=vim`。
全局和局部参数配置用户信息，略。