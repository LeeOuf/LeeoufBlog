---

title: 开发笔记：Flutter
date: 2019-03-11 17:10:27
categories: 笔记
tags: 
    - Flutter 
    - 笔记
---

Flutter官方实例Gallery代码的阅读笔记。
Flutter选择的dart语言方法名使用的是全小写下划线的规则，而类名命名则是驼峰。
(dart命名规范)[https://www.dartlang.org/guides/language/effective-dart/style]

# main.dart
app入口，fullter的runApp方法如下
```
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```
`..`是dart中的级联操作符，表示连续调用方法，WidgetsFlutterBinding继承于BindingBase，ensureInitialized方法很简单，创建了一个自身的实例。
接下来attachRootWidget方法，flutter中有一个很重要的Widget概念，从app到一个按钮全都是Widget，而这些Widget与H5中的doom一样，构成了一个Widget树，attachRootWidget方法就获得了最顶层的根节点。
最后的scheduleWarmUpFrame方法，里面有两个Timer，根据方法名看在做一些绘制的工作。

# app.dart
GalleryApp中有不少属性，除了几个bool类型的外，updateUrlFetcher用于App更新提醒（可以借鉴一下用于灰度testFlight），onSendFeedback用于反馈。
_GalleryAppState中的GalleryOptions存放夜间模式、文字方向、字体大小等全局定义（这部分在自己开发时可以做成本地缓存或server下发，拉到数据之后映射到options）。AppStateModel其实很类似，用于购物车业务。
在build方法中初始化GalleryHome，返回了一个ScopedModel StatelessWidget，指定home。

