---
title: Runtime：load和initialize
date: 2019-06-07 14:02:55
categories: Runtime
tags:
    - load
    - initialize
---
[Load和Initialize往死了问是一种怎样的体验？](https://www.jianshu.com/p/bd82ef5ea186)这篇blog有很好的参考价值，不过有一些细节仍然存在错误。

`+ load()`方法，顾名思义是类的加载方法，在`main()`函数之前调用，其官方文档如下：
```
Invoked whenever a class or category is added to the Objective-C runtime; implement this method to perform class-specific behavior upon loading.

当类（Class）或者类别（Category）加入Runtime中时（就是被引用的时候）。
实现该方法，可以在加载时做一些类特有的操作。

Discussion

The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.

The order of initialization is as follows:

All initializers in any framework you link to.
调用所有的Framework中的初始化方法

All +load methods in your image.
调用所有的+load方法

All C++ static initializers and C/C++ attribute(constructor) functions in your image.
调用C++的静态初始化方及C/C++中的attribute(constructor)函数

All initializers in frameworks that link to you.
调用所有链接到目标文件的framework中的初始化方法

In addition:

A class’s +load method is called after all of its superclasses’ +load methods.
一个类的+load方法在其父类的+load方法后调用

A category +load method is called after the class’s own +load method.
一个Category的+load方法在被其扩展的类的自有+load方法后调用

In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.
在+load方法中，可以安全地向同一二进制包中的其它无关的类发送消息，但接收消息的类中的+load方法可能尚未被调用。
```

`+ initialize()`方法，顾名思义即类的初始化方法，其官方文档如下：
```
Initializes the class before it receives its first message.

在这个类接收第一条消息之前调用。

Discussion

The runtime sends initialize to each class in a program exactly one time just before the class, or any class that inherits from it, is sent its first message from within the program. (Thus the method may never be invoked if the class is not used.) The runtime sends the initialize message to classes in a thread-safe manner. Superclasses receive this message before their subclasses.

Runtime在一个程序中每一个类的一个程序中发送一个初始化一次，或是从它继承的任何类中，都是在程序中发送第一条消息。（因此，当该类不使用时，该方法可能永远不会被调用。）运行时发送一个线程安全的方式初始化消息。父类的调用一定在子类之前。
```

其基础特征及调用时序可以在文档中有一个大概的了解，然后我们在这个基础上提出问题。
前提：A类、B类，均实现其load及initialize方法，B为A的子类，另外C1、C2为A的分类。
1. 不做任何操作，求时序。
2. 对A类发消息，求时序。
3. 对B类发消息，求时序。
此后问题删除分类。
4. 先对A类发消息，再对B类发消息，求时序。
5. 先对B类发消息，再对A类发消息，求时序。
6. 删除B类initialize方法，先对A类发消息，再对B类发消息，求时序。
7. 在A类的load方法中对B类发消息，此外对A类发消息，求时序。
8. 取消B类和A类的继承关系，改为继承NSObject，条件同问题7，求时序。

输入如下：
```
问题1：
2019-06-07 14:25:28.996053+0800 LKTestOC[59335:5384227] A: father load
2019-06-07 14:25:28.996694+0800 LKTestOC[59335:5384227] B: son load
2019-06-07 14:25:28.996812+0800 LKTestOC[59335:5384227] C1: category load
2019-06-07 14:25:28.996881+0800 LKTestOC[59335:5384227] C2: category load
结论：load方法时序 父类->子类->分类，不会覆盖，且与消息发送无关。

问题2：
2019-06-07 14:26:33.960364+0800 LKTestOC[59510:5385993] A: father load
2019-06-07 14:26:33.960938+0800 LKTestOC[59510:5385993] B: son load
2019-06-07 14:26:33.961009+0800 LKTestOC[59510:5385993] C1: category load
2019-06-07 14:26:33.961078+0800 LKTestOC[59510:5385993] C2: category load
2019-06-07 14:26:34.069072+0800 LKTestOC[59510:5385993] C2: category initialize
结论：initialize方法在消息发送后调用，会覆盖，分类时序最后，且与主类是否import分类无关。

问题3：
2019-06-07 14:32:15.164806+0800 LKTestOC[60397:5393459] A: father load
2019-06-07 14:32:15.165367+0800 LKTestOC[60397:5393459] B: son load
2019-06-07 14:32:15.165457+0800 LKTestOC[60397:5393459] C1: category load
2019-06-07 14:32:15.165530+0800 LKTestOC[60397:5393459] C2: category load
2019-06-07 14:32:15.269258+0800 LKTestOC[60397:5393459] C2: category initialize
2019-06-07 14:32:15.269361+0800 LKTestOC[60397:5393459] B: son initialize
结论：initialize方法时序 父类->分类->子类，分类覆盖父类。

问题4、5：
2019-06-07 14:33:54.319935+0800 LKTestOC[60679:5396081] A: father load
2019-06-07 14:33:54.320518+0800 LKTestOC[60679:5396081] B: son load
2019-06-07 14:33:54.466408+0800 LKTestOC[60679:5396081] A: father initialize
2019-06-07 14:33:54.466562+0800 LKTestOC[60679:5396081] B: son initialize
结论：子类的initliaze会自动调用父类方法，且每个类初始化时只会调用一次initliaze。

问题6：
2019-06-07 14:38:39.735874+0800 LKTestOC[61421:5403318] A: father load
2019-06-07 14:38:39.736519+0800 LKTestOC[61421:5403318] B: son load
2019-06-07 14:38:39.921747+0800 LKTestOC[61421:5403318] A: father initialize
2019-06-07 14:38:39.921889+0800 LKTestOC[61421:5403318] A: father initialize
结论：如官方文档所说，子类未实现intialize时父类会调用多次，此处要重点注意，如果想利用initialize做懒加载，需防止调用多次，可利用(self == [ClassName self])做判断。

问题7：
2019-06-07 14:43:10.949354+0800 LKTestOC[62160:5409931] A: father initialize
2019-06-07 14:43:10.949928+0800 LKTestOC[62160:5409931] B: son initialize
2019-06-07 14:43:11.008578+0800 LKTestOC[62160:5409931] A: father load
2019-06-07 14:43:11.008811+0800 LKTestOC[62160:5409931] B: son load
结论：initialize方法不一定在main()之后，严格遵循发送消息时调用。

问题8：
2019-06-07 14:44:12.102702+0800 LKTestOC[62327:5411529] B: son initialize
2019-06-07 14:44:12.105044+0800 LKTestOC[62327:5411529] A: father load
2019-06-07 14:44:12.105306+0800 LKTestOC[62327:5411529] B: son load
结论：符合预期。
```
