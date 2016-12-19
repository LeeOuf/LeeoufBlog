---
title: 笔记：iOS开发过程中遇到的坑
date: 2016-12-19 15:48:43
categories: iOS
tags: 
    - 笔记
---

1. NSNumber类型判断是否为0不能使用`aIntegerNum == 0`，而应使用`[aIntegerNum isEqualToNumber:@0]`，前者比较地址，后者比较值。
2. UITableView无数据源时，numberOfSections是1，但[tableView numberOfRowsInSection:0]是0。
3. 时间戳转换时间时会相差8小时，需要计算时区。
4. 