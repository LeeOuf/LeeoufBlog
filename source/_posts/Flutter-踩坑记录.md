---
title: Flutter-踩坑记录
date: 2019-04-10 14:59:08
categories: Flutter
tags:
    - Flutter
    - 笔记
    - 踩坑记录
---

# Flutter 踩坑记录

## 问题索引

[TabBar及PageView状态保存](#q001)
[ScrollView滚动进度保存](#q002)
[ListView下拉刷新后数据已返回，UI未更新](#q003)
[中文汉字变形](#q004)
[热重载hot reload未生效](#q005)
[转场动画在list内黑屏不生效](#q006)
[Android plugin内不能使用AAR包](#q007)
[Android plugin内AAR包内的so库加载不出来](#q008)
[Json注解制动生成的解析文件解析数据失败](#q009)
[加载更多后头像加载慢或不加载](#q010)
[dio安卓上网络请求失败](#q011)
[NestedScrollView中的子View使用controller监听会导致Sliver控件吸顶效果失效](#q012)
[枚举值需要取index，否则打印类名](#q013)
[flutter动画不执行](#q014)
[在SilverAppBar内使用滑动控件（ScrollView、PageView等）滑动控件消失](#q015)
[dispose后不允许setState](#q016)
[滑动冲突](#q017)
[Row中的Text控件出现overflow警告](#q018)
[webview无法连接网络](#q019)

## q001

**TabBar及PageView状态保存**

*Flutter 1.2.1 statble*

子页面离开屏幕时，state会被销毁引发的bug
https://juejin.im/post/5b73c3b3f265da27d701473a

## q002

**ScrollView滚动进度保存**

*Flutter 1.2.1 statble*

进入其他页面返回后保存scrollView的offset
https://stackoverflow.com/questions/49087703/preserving-state-between-tab-view-pages

## q003

**ListView下拉刷新后数据已返回，UI未更新**

*Flutter 1.2.1 statble*

ListView需要有Key，否则被复用引发的UI不立即更新bug
https://stackoverflow.com/questions/51279611/flutter-listview-builder-not-updating-after-insert

## q004

**中文汉字变形**

*Flutter 1.2.1 statble*

该问题发生在MaterialApp中，多数情况与fontfamily有关。
https://github.com/flutter/flutter/issues/22966
https://github.com/flutter/flutter/issues/26752
https://github.com/flutter/flutter/issues/25726

## q005

**热重载hot reload未生效**

*Flutter 1.2.1 statble*

目前已知几种情况下，hot reload无法生效

- 需要build_runner做代码生成部分的代码
- 使用了新添加的资源

发现hot reload无效，可以尝试使用hot restart重新启动应用，若还是无效，可以考虑执行ScriptForCI/update.sh后，重新flutter run。

## q006
**转场动画在list内黑屏不生效**
Hero转场动画在list内黑屏不生效
原因：list内可能存在多个元素，Hero tag固定的情况下会判定view tree里面存在多个相同tag的转场元素
解决方案：tag随机生成，并通过参数传入下个页面

## q007
**Android plugin内不能使用AAR包**
Android插件开发中，AAR包引用不到
原因：flutter暂不支持AAR的引用，仅支持gradle方式引用
解决方案：
1. 放弃插件方案，直接在application工程下开发flutter与native的交互方案
2. 百度私服。将AAR文件放入百度私服maven库，通过gradle方式引用。
WIKI：http://wiki.baidu.com/pages/viewpage.action?pageId=465488650
maven库地址：http://maven.baidu-int.com/nexus/content/groups/public
链接：http://maven.baidu-int.com/nexus/content/groups/public/com/baidu/bdpass/
使用方法：compile 'com.baidu.bdpass:HWOpenSDK:3.3.3@aar'
接口人：zhanghuaming@baidu.com

## q008
**Android plugin内AAR包内的so库加载不出来**
Android 插件通过gradle方式引用AAR包，但是其中的so库load不出来
原因：flutter对AAR包的引用支持不好
解决方案：放弃使用pass提供的AAR包，用贴吧主端的sofire-sdk-3.1.9.3.jar + libfire.so替代

## q009
**Json注解制动生成的解析文件解析数据失败**
原因：server返回的数据有问题，一个map的结构体可能返回list类型的空串
解决方案：在解析之前对可能发生错误的地方做预处理，如果类型不对直接置null

## q010
**Json注解制动生成的解析文件解析数据失败**
原因：View复用导致没有刷新
解决方案：需要给头像控件增加一个单独的Key

##q011
原因：没有网络请求的权限
解决方案：在tbflutterlite/android/app/src/main/AndroidMainifest.xml中添加网络请求权限，同理如果需要读写SD卡也需要声明对应的权限

##q012
原因：未知
解决方案：改用NotificationListener

## q014
**flutter动画不执行**
原因：flutter TickerProvider同一时间只能对一个动画生效，且动画执行完一次后，下次执行时起始数据在末尾
方案：动画controller写成全局变量，执行动画前先调reset方法。

## q015
**在SilverAppBar内使用滑动控件（ScrollView、PageView等）滑动控件消失**
原因不明。

``` dart
SliverAppBar(
  flexibleSpace: FlexibleSpaceBar(
    background: PageView.builder(
      ... ...
    )
  )
)

```

PageView若更换成其他非滑动型控件，则无此问题。

## q016
原因：内存泄漏
方案：dispose时将callBack等异步操作取消或置null

## q017
**滑动冲突**
原因：flutter事件传递机制是由外向内传递的，外层如果处理了事件，会直接resume掉事件流，内层控件没有机会处理事件
解决方案：对于GestureDetector onScale 和 onDrag事件不能共存的问题，通过两层嵌套的GestureDetector解决。对于系统外层控件直接处理滑动事件的情况（例如tabView），通过内层控件回调外层标志位决定外层是否处理事件，例如
```
TabBarView(
   children: children,
   physics: _needHandleScroll ? PageScrollPhysics() : NeverScrollableScrollPhysics(),
),
```

## q018
方案：将Text嵌套在Expanded中

## q019
Info.plist中加入
```
<key>io.flutter.embedded_views_preview</key>
<true/>
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```
