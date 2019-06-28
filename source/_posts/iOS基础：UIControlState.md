---
title: iOS基础：UIControlState
date: 2016-10-08 19:40:13
categories: iOS基础知识
tags:
    - UIControlState
---

以UIButton为例，我们为button的UIControlStateSelected状态设置一个实心五角星，为UIControlStateNormal状态设置一个空心五角星，最后为其设置UIControlEventTouchUpInside事件的action。

在一些业务场景中，我们需要为其设置一个点击态（按下但没有松开手指）时的样式，该如何做呢？

看一下UIControlState的定义：
```
typedef NS_OPTIONS(NSUInteger, UIControlState) {
    UIControlStateNormal       = 0,
    UIControlStateHighlighted  = 1 << 0,                  // used when UIControl isHighlighted is set
    UIControlStateDisabled     = 1 << 1,
    UIControlStateSelected     = 1 << 2,                  // flag usable by app (see below)
    UIControlStateFocused NS_ENUM_AVAILABLE_IOS(9_0) = 1 << 3, // Applicable only when the screen supports focus
    UIControlStateApplication  = 0x00FF0000,              // additional flags available for application use
    UIControlStateReserved     = 0xFF000000               // flags reserved for internal framework use
};
```

**需要注意，这是一个NS_OPTIONS类型，而不是NS_ENUM**
这几种状态转化成二进制实际上是：
```
   0
   1
  10
 100
1000
```
这意味着我们可以对其进行位操作，比如：
`[button setImage:selectedImage forState:UIControlStateSelected | UIControlStateHighlighted];`
以上表示，设置按钮被选中且高亮时的图片。