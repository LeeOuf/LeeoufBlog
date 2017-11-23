---
title: iOS开发过程中遇到的诸多问题
date: 2016-12-19 15:48:43
categories: 笔记
tags: 
    - iOS
    - 笔记
---

1. NSNumber类型判断是否为0不能使用`aIntegerNum == 0`，而应使用`[aIntegerNum isEqualToNumber:@0]`，前者比较地址，后者比较值。
2. UITableView无数据源时，numberOfSections是1，但[tableView numberOfRowsInSection:0]是0。
3. 时间戳转换时间时会相差8小时，需要计算时区。
4. iOS10中UISwitch的特异性
5. 快速便利Array\Set时，对元素进行操作必挂
6. 在block中使用成员变量，会导致循环引用
7. iOS10以下导航栏问题[http://www.jianshu.com/p/e4448c24d900](导航栏隐藏 && 导航栏错乱) [http://www.jianshu.com/p/4b8af425a7d0](iOS 隐藏导航栏导致导航栏错乱的那些坑)
8. __block相当于unsafe_unretain，在某些情况下，会导致crash
```
self.string1 = @"String 1";   
self.string2 = self.string1;   // unsafe_unretain
self.string1 = nil;  
NSLog(@"String 2 = %@", self.string2);  
```
