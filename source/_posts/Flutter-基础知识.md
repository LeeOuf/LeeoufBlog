---
title: Flutter-基础知识
date: 2019-06-11 13:37:30
categories: Flutter
tags:
    - Flutter
---
# Flutter的架构
{% asset_img 架构.jpeg Flutter架构 %}
- Framework：日常开发中直接接触的一层，包括UI组件、动画等。
- Engine：基于C/C++的引擎，包括绘制引擎skia，Dart VM，Platform Channel。
- Embedder：嵌入层，将Flutter嵌入到各平台。做原生Plugin、线程管理等。

# 线程模型
{% asset_img 线程模型.png 线程模型 %}
参考
[Flutter Engine线程管理与Dart Isolate机制](https://www.jianshu.com/p/aaa6a8b1d6b0)
[Flutter Engine 线程模型](https://zhuanlan.zhihu.com/p/64034467)

# Widget生命周期
{% asset_img Widget生命周期.png Widget生命周期 %}
参考[flutter中的生命周期](https://segmentfault.com/a/1190000015211309?utm_source=tag-newest)


