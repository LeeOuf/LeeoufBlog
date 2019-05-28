---
title: iOS基础知识面归纳
date: 2019-05-27 16:58:29
categories: iOS基础知识
tags:
    - iOS基础知识
---
对这几年来开发中遇到的常见基础知识点做一个总结。

# 启动优化相关
参考[iOS-Block的本质](https://www.jianshu.com/p/4e79e9a0dd82)
[关于Block用copy修饰的原因的一点自己的理解](https://www.jianshu.com/p/bf3798fe3f49)
## load方法和initialize方法的异同
load方法官方文档如下：
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
initialize官方文档如下：
```
Initializes the class before it receives its first message.

在这个类接收第一条消息之前调用。

Discussion

The runtime sends initialize to each class in a program exactly one time just before the class, or any class that inherits from it, is sent its first message from within the program. (Thus the method may never be invoked if the class is not used.) The runtime sends the initialize message to classes in a thread-safe manner. Superclasses receive this message before their subclasses.

Runtime在一个程序中每一个类的一个程序中发送一个初始化一次，或是从它继承的任何类中，都是在程序中发送第一条消息。（因此，当该类不使用时，该方法可能永远不会被调用。）运行时发送一个线程安全的方式初始化消息。父类的调用一定在子类之前。
```

可以从官方文档总结出，load方法会在Runtime中（应用启动时）被调用，initialize是在类接收第一条消息时被调用，两者都是先调用本类，再调用类别，两者均自动调用父类，不需要super操作，且自动调用仅一次。

# Block
## Block本质
Block是一种OC对象，内部有isa指针，它封装了函数调用和函数调用环境的OC对象。它会捕获变量的临时值（遇到过一个BUG，Block初始化时捕获了变量值，后续每一次调用时本应该基于最新的值来做业务），若想要在block内部改变外部值，使用__block。

## Block类型
- __NSGlobalBlock __ （ _NSConcreteGlobalBlock ） 数据区
- __NSStackBlock __ （ _NSConcreteStackBlock ） 堆区
- __NSMallocBlock __ （ _NSConcreteMallocBlock ） 栈区

## 如何判断block是哪种类型
- 没有访问auto变量的block是__NSGlobalBlock __ ，放在数据段
- 访问了auto变量的block是__NSStackBlock __
- [__NSStackBlock __ copy]操作就变成了__NSMallocBlock __
因此
- __NSGlobalBlock __调用copy操作后，什么也不做
- __NSStackBlock __ 调用copy操作后，复制效果是：从栈复制到堆；副本存储位置是堆
- __NSStackBlock __ 调用copy操作后，复制效果是：引用计数增加；副本存储位置是堆

## ARC下Block何时自动复制到堆上
- block作为函数返回值时
- 将block赋值给__strong指针时
- block作为Cocoa API中方法名含有usingBlock的方法参数时
- block作为GCD API的方法参数时

## 为什么用copy修饰
栈区Block在MRC下不会像ARC中那样自动copy，因此栈区的Block容易在方法执行完后自动释放导致野指针crash。

## Block的内存泄漏
Block最典型的循环引用就是self持有block，block持有self，为了避免循环引用，通常使用__weak或__block的弱引用，在此基础上，还衍生出了weak strong dance，来避免block内部引用对象被释放导致的野指针crash或bug。

# Runtime
参考[iOS运行时(Runtime)详解+Demo](https://www.jianshu.com/p/adf0d566c887)
## Runtime本质
OC是一门由C和汇编语言写的面向对象的动态语言。关键在于理解Class的数据结构定义。
```
typedef struct object_class *Class;
struct object_class{
    Class isa OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
     Class super_class                        OBJC2_UNAVAILABLE;  // 父类
     const char *name                         OBJC2_UNAVAILABLE;  // 类名
     long version                             OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
     long info                                OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
     long instance_size                       OBJC2_UNAVAILABLE;  // 该类的实例变量大小
     struct objc_ivar_list *ivars             OBJC2_UNAVAILABLE;  // 该类的成员变量链表
     struct objc_method_list *methodLists     OBJC2_UNAVAILABLE;  // 方法定义的链表
     struct objc_cache *cache                 OBJC2_UNAVAILABLE;  // 方法缓存
     struct objc_protocol_list *protocols     OBJC2_UNAVAILABLE;  // 协议链表
#endif
}OBJC2_UNAVAILABLE;

typedef struct objc_object *id;
struct objc_object{
     Class isa OBJC_ISA_AVAILABILITY;
};

typedef struct objc_category *Category
struct objc_category{
     char *category_name                         OBJC2_UNAVAILABLE; // 分类名
     char *class_name                            OBJC2_UNAVAILABLE;  // 分类所属的类名
     struct objc_method_list *instance_methods   OBJC2_UNAVAILABLE;  // 实例方法列表
     struct objc_method_list *class_methods      OBJC2_UNAVAILABLE; // 类方法列表
     struct objc_protocol_list *protocols        OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}
```

## SEL和IMP的区别
SEL的数据结构：`typedef struct objc_selector *SEL；`，它是指向一个方法的指针；
IMP的定义：`id (*IMP)(id, SEL,...)`，它是一个函数指针，指向方法实现的地址。第一个参数：是指向self的指针(如果是实例方法，则是类实例的内存地址；如果是类方法，则是指向元类的指针)
第二个参数：是方法选择器(selector)

## Method
```
typedef struct objc_method *Method
struct objc_method{
    SEL method_name      OBJC2_UNAVAILABLE; // 方法名
    char *method_types   OBJC2_UNAVAILABLE;
    IMP method_imp       OBJC2_UNAVAILABLE; // 方法实现
}
```
Method可以理解为方法名和方法实现的map映射，便于我们通过方法指针找到方法实现。OC的消息转发、方法动态绑定、方法交换都是基于这个机制。

# RunLoop
参考[iOS底层原理总结-RunLoop](https://www.jianshu.com/p/de752066d0ad)
## RunLoop本质
就是一个while循环......源码如下：
```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;          /* locked for accessing mode list */
    __CFPort _wakeUpPort;           // used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};

typedef struct __CFRunLoopMode *CFRunLoopModeRef;
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;  /* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

## RunLoop作用
- 保证主线程不被销毁（写过那么多hello world应该知道main函数顺序执行完就退出了，很好理解）。
- 处理用户事件，传感器、通知。
- 调度CPU资源，让我们在空闲时能干很多事。

## RunLoop几种状态
```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
     kCFRunLoopEntry = (1UL << 0),   //   即将进入RunLoop
     kCFRunLoopBeforeTimers = (1UL << 1), // 即将处理Timer
     kCFRunLoopBeforeSources = (1UL << 2), // 即将处理Source
     kCFRunLoopBeforeWaiting = (1UL << 5), //即将进入休眠
     kCFRunLoopAfterWaiting = (1UL << 6),// 刚从休眠中唤醒
     kCFRunLoopExit = (1UL << 7),// 即将退出RunLoop
     kCFRunLoopAllActivities = 0x0FFFFFFFU
     };
```

# 内存管理相关
## ARC
### ARC的本质
ARC本质上和MRC没有区别，都是依靠引用计数进行管理，只是编译器帮我们做了内存管理的操作。ARC有4种修饰符__strong,__weak,__autoreleasing,__unsafe_unretained。

### 内存泄漏
内存泄漏最常出现在循环引用，表现为A强引用B，B强引用A的循环，会导致双方的引用计数不可能为0，无法释放，内存无法释放的结果就是app占用内存逐渐扩大，最终被看门狗杀死进程crash。

### 野指针
Crash中出现频率较高的BAD_ACCESS就是野指针Crash之一，即指针指向的内存已经在别处被回收。通常出现在MRC或者iOS9以前的ARC上，常见的如block（见过最多的就是把block从通知中抛出给别的地方处理，省事的代码容易挖坑）等。调试方式主要依靠XCode的Zoombie Object和Address Sanitizer。

### Autoreleasepool
参考 [Autoreleasepool](https://www.jianshu.com/p/a2999d7728b4)
[AutoreleasePool底层实现原理](https://www.jianshu.com/p/50bdd8438857)
- AutoreleasePool创建是在一个RunLoop事件开始之前(push)
- AutoreleasePool释放是在一个RunLoop事件即将结束之前(pop)。
- AutoreleasePool里的Autorelease对象的加入是在RunLoop事件中，AutoreleasePool里的Autorelease对象的释放是在AutoreleasePool释放时。

### MRC怎么写
像init，copy这些实例方法，是由对象持有者管理内存的，所以在MRC中要主动release，而stringWithFormat之类的类方法则是由类自身去管理。基础原则即：谁创建谁释放。

# UIKit
## UITableView调优
### 系统API
- Cell复用池
- 预估高度

### TableViewKit封装
封装tableview、datasource、delegate等，提供：
- 二维数组数据源。
- Cell内部高度计算类方法。
- 上下拉刷新、左右滑动。

### 内容优化
- 减少主线程操作，异步加载，如图片。
- 异步绘制，如YYTextLabel。
- 减少对象创建，创建好，改变hidden。
- 减少属性赋值，frame的改变会引发重绘，在android端有一个实践，把Cell拆分成N个小Cell，在直觉上这是违背优化的，但是带来了非常可观的性能优化，我认为当业务到达一定复杂度阈值的时候，将Cell拆分可以减少频繁赋值导致的性能开销。
- 减少离屏渲染，如masks\shadow\corner等，尽量用hidden避免用alpha。
- 谨慎使用autolayout。

### Layouter
Layouter与MVVM思想类似，将server端的数据转化为与View绑定的ViewModel（整理上游数据，保证View能用）。一次计算好布局数据后就可以避免在heightForRow和cellForRow-bindData过程中重复计算。再进一步也可以作为下次更新的缓存，用于预估高度、减少白屏时间等。缓存高度最好是在runloop空闲时，参考SDWebImage。

### 性能检测工具
- profile instruments
- 代码打点

# 多线程
## 概念
多线程就是说可以进行多个任务并发，通常线程数等于物理核心数，在后来有了四核八线程等基于逻辑核心的超线程技术。

## GCD
参考[iOS 多线程：『GCD』详尽总结](https://www.jianshu.com/p/2d57c72016c6)
GCD有两个核心概念，队列和任务，重点在理解串行、并发队列；主线程、子线程；同步执行、异步执行。
除了常见的同步、异步dispatch外，还有dispatch_barrier_async（栅栏）、dispatch_after（延时）、dispatch_once（只执行一次）、dispatch_apply（迭代）、dispatch_group（组）、dispatch_group_notify、dispatch_group_wait、dispatch_group_enter、dispatch_group_leave等。

## dispatch_barrier和dispatch_group的区别
参考[dispatch_barrier_async和dispatch_barrier_sync的区别和详细解析](https://www.jianshu.com/p/a0ce5e51286d)
dispatch_barrier_async和sync的区别：
相同点：
- 等待在它前面插入队列的任务先执行完
- 等待他们自己的任务执行完再执行后面的任务
不同点：
- dispatch_barrier_sync将自己的任务插入到队列的时候，需要等待自己的任务结束之后才会继续插入被写在它后面的任务，然后执行它们。
- dispatch_barrier_async将自己的任务插入到队列之后，不会等待自己的任务结束，它会继续把后面的任务插入到队列，然后等待自己的任务结束后才执行后面任务。

## NSOperation
CCD基于C，NSOperation基于OC，其底层也是通过GCD实现，NSOperation比GCD更抽象，API更丰富，效率较低，对于绝大多数的业务需求来说GCD已经完全够用，所以我对NSOperation的了解也在皮毛而已。

# 缓存
## iOS数据持久化方案
1. 沙盒 - 每个应用程序对应的系统目录
1.1. plist
1.2. SQLite FMDatabase
1.3. NSKeyedArchiver 归档成文件 NSFileManager
2. CoreData

## iOS缓存
1. NSCache
2. NSURLCache
3. 内存缓存

# 网络
## TCP和UDP
TCP有三次握手，可靠传输，UDP是不可靠传输。TCP的滑动窗口是接收方为了流量控制限制的窗口大小，控制发送速度防止自己被淹没。

## Https，SSL
Http是无状态的，而Https是加入了SSL层，可进行加密传输、身份认证的网络协议。

## cookie和session
cookie和session的最大区别就是cookie保存在客户端，session保存在服务端。

# 散乱知识点
1. [KVO、Delegate、Notification 区别及相关使用场景](https://www.jianshu.com/p/9215251693f0)