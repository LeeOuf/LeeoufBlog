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
9. MRC goto语法，目标语句中的代码，与switch中的finally类似，正常执行。
10. A页面跳B页面，iOS11先走B页面的viewDidLoad再走A页面的viewWillDisappear，iOS10先走A页面的viewWillDisappear再走B页面的viewDidLoad。
11. iOS10及以下系统，在viewDidLoad中设置导航栏标题的透明度是无效的，会被系统重设；而在iOS11中是生效的，但是，在滑动返回过程中，alpha会被重设为1。
12. Notification在主线程中是同步执行的。
13. 若给UIButton高亮状态设置了backgroundImage，则需要相应地设置高亮状态icon及字色。