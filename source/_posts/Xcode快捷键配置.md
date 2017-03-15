---
title: Xcode快捷键配置
date: 2016-09-21 10:12:12
categories: 笔记
tags:
    - 开发环境配置
    - Xcode
---
部分内容来自：[简书：为Xcode添加删除当前行、复制当前行快捷键](http://www.jianshu.com/p/2ed2c7ac6d53?nomobile=yes) 感谢作者的分享。
本文仅为记录个人的开发配置。

Xcode8以后，不再支持plugins，因此之前用来设置快捷键的boost等插件都已失效。
有一些eclipse带过来的快捷键习惯该如何设置呢？
<!-- more -->

# 上下移动当前行
Xcode -> Preference -> Key Bindings -> Move Line Up & Move Line Down

# 复制当前行
## 修改权限
```
sudo chmod 666 /Applications/Xcode.app/Contents/Frameworks/IDEKit.framework/Resources/IDETextKeyBindingSet.plist

sudo chmod 777 /Applications/Xcode.app/Contents/Frameworks/IDEKit.framework/Resources/
```
## 添加快捷键
```
open /Applications/Xcode.app/Contents/Frameworks/IDEKit.framework/Resources/IDETextKeyBindingSet.plist
```
在`Insertions and Indentations`中添加`Duplicate Current Line` - `selectLine:, copy:, moveToEndOfLine:, insertNewline:, paste:, deleteBackward:`
## 设置快捷键
重启后
Xcode -> Preference -> Key Bindings -> Insertions and Indentations

# 删除当前行
## 添加快捷键
```
open /Applications/Xcode.app/Contents/Frameworks/IDEKit.framework/Resources/IDETextKeyBindingSet.plist
```
在`Deletions`中添加`Delete Current Line` - `deleteToBeginningOfLine:, moveToEndOfLine:, deleteToBeginningOfLine:, deleteBackward:, moveDown:, moveToBeginningOfLine:`
## 设置快捷键
重启后
Xcode -> Preference -> Key Bindings -> Delete Current Line
