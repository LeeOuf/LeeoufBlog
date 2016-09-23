---
title: 'iOS基础: 类、对象和方法'
date: 2016-08-14 16:15:39
categories: iOS
tags:
    - iOS
    - 程序设计基础
    - 类
    - 对象
    - 方法
    - autoreleasepool
---

本文是在阅读`Objective-C程序设计  第6版`一书过程中写的学习笔记，文中出现的大部分代码基本与书中相同。

# 什么是对象
我们知道，OC是在C的基础上设计的`面向对象`的程序设计语言，而C则是`过程性语言`。
以我的理解，所谓的过程性语言，是为了实现某种功能所设计的，比如我想要将两个整数累加，返回它们的和，这是我的目的，所以C语言中完成这种目的一段代码称为函数，这个函数的功能是累加两个整数。
而所谓的面相对象，这个函数则被看作一个真实存在的物件，这里我们叫它计算器，当我们想要累加两个整数时，创建了一个计算器对象，然后再调用它本身的功能（方法）来实现我们的目的。

我认为可以这样理解，过程性语言中，“我”是主体，当“我”想要做什么时，创建了“我”需要的函数然后去使用它；而面向对象，每个对象都是独立存在的主体，它们可以为“我”服务。

# 类、方法
**Fraction类**
```
#import <Foundation/Foundation.h>

@interface Fraction : NSObject

- (void)print;
- (void)setNumerator: (int)n;
- (void)setDenominator: (int)d;

@end
```

```
#import "Fraction.h"

@implementation Fraction {
    int numerator;
    int denominator;
}

- (void)print {
    NSLog(@"%i / %i", numerator, denominator);
}

- (void)setNumerator:(int)n {
    numerator = n;
}

- (void)setDenominator:(int)d {
    denominator = d;
}
@end
```

**main函数**
```
#import <Foundation/Foundation.h>
#import "Fraction.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        Fraction *fraction;
        
        fraction = [[Fraction alloc] init];
        [fraction setNumerator:1];
        [fraction setDenominator:3];
        
        NSLog(@"The value of fracton is: ");
        [fraction print];
    }
    return 0;
}
```

先看Fraction类，这里我们只关注它的头文件，这个对象告诉我们，它的功能是通过分子和分母获得一个分数，它有3个对外部可见的方法，打印分数、获得分子、获得分母。这里我们可以完全不必理会它的实现，只需要获知并且相信它很好地实现了这三个功能。
另外@符号在""前，表示这是常量NSString对象。

然后来看main函数。
第一行中，我们就看到了`@autoreleasepool`，这是什么呢？
## autoreleasepool
嵌套在其后的代码块会被放在“自动释放池”这一语境中执行。
什么是“自动释放池”？首先我们需要知道什么是Runloop，Runloop是一个while循环，当然这是一个很复杂的循环。可以理解为一个App中有许多个Runloop，它们在监听着用户操作、管理着内存以及执行一些操作。以后我们再来关心Runloop的具体实现。

[对于每一个Runloop， 系统会隐式创建一个Autorelease pool，这样所有的release pool会构成一个象CallStack一样的一个栈式结构，在每一个Runloop结束时，当前栈顶的Autorelease pool会被销毁，这样这个pool里的每个Object会被release。](http://blog.sina.com.cn/s/blog_8c87ba3b0100tgfs.html)

以下对于autoreleasepool的分析参考了[@雷纯锋的技术博客](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)
文中的实验在部分新设备上已经不再成立，因此结合自己的想法重新做一遍测试。

首先，从上文我们可以知道，autoreleasepool就是延迟调用了realease，那么我们来实验一下它到底是什么时候realease的。
### 场景1
```
__weak NSString *string_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 1.1
    NSString *string = [NSString stringWithFormat:@"I'm here!"];
    NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
    string_weak_ = string;

    NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
    NSLog(@"string_weak_@viewDidLoad: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    NSLog(@"string_weak_@viewWillAppear: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    NSLog(@"string_weak_@viewDidAppear: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
}
```

```
__strong NSString *string_strong_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 1.2
    NSString *string = [NSString stringWithFormat:@"I'm here!"];
    NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
    string_strong_ = string;

    NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
    NSLog(@"string_strong_@viewDidLoad: %@ with retainCount: %@", string_strong_, [string_strong_ valueForKey:@"retainCount"]);
}

// ...省略
```

```
__weak Fraction *fraction_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 1.3
    Fraction *fraction = [[Fraction alloc] init];
    NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
    fraction_weak_ = fraction;

    NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
    NSLog(@"fraction_weak_@viewDidLoad: %@ with retainCount: %@", fraction_weak_, [fraction_weak_ valueForKey:@"retainCount"]);
}
```

```
__strong Fraction *fraction_strong_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 1.4
    Fraction *fraction = [[Fraction alloc] init];
    NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
    fraction_strong_ = fraction;

    NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
    NSLog(@"fraction_strong_@viewDidLoad: %@ with retainCount: %@", fraction_strong_, [fraction_strong_ valueForKey:@"retainCount"]);
}
```

```
// 场景1.1(__weak NSString *) 结果 
2016-08-15 14:26:03.193 LKTestAutoreleasePool[36540:6293792] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 14:26:03.194 LKTestAutoreleasePool[36540:6293792] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 14:26:03.194 LKTestAutoreleasePool[36540:6293792] string_weak_@viewDidLoad: I'm here! with retainCount: 4
2016-08-15 14:26:03.194 LKTestAutoreleasePool[36540:6293792] string_weak_@viewWillAppear: I'm here! with retainCount: 3
2016-08-15 14:26:03.198 LKTestAutoreleasePool[36540:6293792] string_weak_@viewDidAppear: (null) with retainCount: (null)

// 场景1.2(__strong NSString *) 结果 
2016-08-15 14:27:06.619 LKTestAutoreleasePool[36567:6295671] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 14:27:06.619 LKTestAutoreleasePool[36567:6295671] string@viewDidLoad: I'm here! with retainCount: 3
2016-08-15 14:27:06.619 LKTestAutoreleasePool[36567:6295671] string_strong_@viewDidLoad: I'm here! with retainCount: 3
2016-08-15 14:27:06.620 LKTestAutoreleasePool[36567:6295671] string_strong_@viewWillAppear: I'm here! with retainCount: 2
2016-08-15 14:27:06.623 LKTestAutoreleasePool[36567:6295671] string_strong_@viewDidAppear: I'm here! with retainCount: 1

// 场景1.3(__weak Fraction *) 结果
2016-08-15 14:31:36.558 LKTestAutoreleasePool[36655:6301737] fraction@viewDidLoad: <Fraction: 0x7fd942f1dfd0> with retainCount: 1
2016-08-15 14:31:36.558 LKTestAutoreleasePool[36655:6301737] fraction@viewDidLoad: <Fraction: 0x7fd942f1dfd0> with retainCount: 1
2016-08-15 14:31:36.559 LKTestAutoreleasePool[36655:6301737] fraction_weak_@viewDidLoad: <Fraction: 0x7fd942f1dfd0> with retainCount: 3
2016-08-15 14:31:36.559 LKTestAutoreleasePool[36655:6301737] fraction_weak_@viewWillAppear: (null) with retainCount: (null)
2016-08-15 14:31:36.564 LKTestAutoreleasePool[36655:6301737] fraction_weak_@viewDidAppear: (null) with retainCount: (null)

// 场景1.4(__strong Fraction *) 结果
2016-08-15 14:32:09.509 LKTestAutoreleasePool[36681:6303031] fraction@viewDidLoad: <Fraction: 0x7fd9e9d9ec90> with retainCount: 1
2016-08-15 14:32:09.509 LKTestAutoreleasePool[36681:6303031] fraction@viewDidLoad: <Fraction: 0x7fd9e9d9ec90> with retainCount: 2
2016-08-15 14:32:09.509 LKTestAutoreleasePool[36681:6303031] fraction_strong_@viewDidLoad: <Fraction: 0x7fd9e9d9ec90> with retainCount: 2
2016-08-15 14:32:09.510 LKTestAutoreleasePool[36681:6303031] fraction_strong_@viewWillAppear: <Fraction: 0x7fd9e9d9ec90> with retainCount: 1
2016-08-15 14:32:09.514 LKTestAutoreleasePool[36681:6303031] fraction_strong_@viewDidAppear: <Fraction: 0x7fd9e9d9ec90> with retainCount: 1
```
> 我们使用了一个全局的 __weak 变量 string_weak_ 来指向它。因为 __weak 变量有一个特性就是它不会影响所指向对象的生命周期，这里我们正是利用了这个特性。 

最初的两个测试使用了NSString对象，但是常量NSString对象存在常量区，不会dealloc影响了测试，于是换成了前文中的分数类进行测试。
场景1.3中fraction_weak_对象被自动添加到当前的autorealeasepool，当viewDidLoad返回时，被回收。

### 场景2
```
__weak NSString *string_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 2.1
    @autoreleasepool {
        NSString *string = [NSString stringWithFormat:@"I'm here!"];
        NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
        string_weak_ = string;
        NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
        NSLog(@"string_weak_@viewDidLoad: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
    }

    NSLog(@"string_weak_@viewDidLoad: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
}
```

```
__weak Fraction *fraction_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 2.2
    @autoreleasepool {
        Fraction *fraction = [[Fraction alloc] init];
        NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
        fraction_weak_ = fraction;
        NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
        NSLog(@"fraction_weak_@viewDidLoad: %@ with retainCount: %@", fraction_weak_, [fraction_weak_ valueForKey:@"retainCount"]);
    }

    NSLog(@"fraction_weak_@viewDidLoad: %@ with retainCount: %@", fraction_weak_, [fraction_weak_ valueForKey:@"retainCount"]);
}
```

```
// 场景2.1(__weak NSString *) 结果
2016-08-15 14:58:39.531 LKTestAutoreleasePool[36863:6327169] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 14:58:39.531 LKTestAutoreleasePool[36863:6327169] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 14:58:39.531 LKTestAutoreleasePool[36863:6327169] string_weak_@viewDidLoad: I'm here! with retainCount: 4
2016-08-15 14:58:39.532 LKTestAutoreleasePool[36863:6327169] string_weak_@viewDidLoad: (null) with retainCount: (null)
2016-08-15 14:58:39.532 LKTestAutoreleasePool[36863:6327169] string_weak_@viewWillAppear: (null) with retainCount: (null)
2016-08-15 14:58:39.536 LKTestAutoreleasePool[36863:6327169] string_weak_@viewDidAppear: (null) with retainCount: (null)

// 场景2.2(__weak Fraction *) 结果
2016-08-15 14:52:03.420 LKTestAutoreleasePool[36795:6319455] fraction@viewDidLoad: <Fraction: 0x7fe429f425e0> with retainCount: 1
2016-08-15 14:52:03.420 LKTestAutoreleasePool[36795:6319455] fraction@viewDidLoad: <Fraction: 0x7fe429f425e0> with retainCount: 1
2016-08-15 14:52:03.420 LKTestAutoreleasePool[36795:6319455] fraction_weak_@viewDidLoad: <Fraction: 0x7fe429f425e0> with retainCount: 3
2016-08-15 14:52:03.421 LKTestAutoreleasePool[36795:6319455] fraction_weak_@viewDidLoad: (null) with retainCount: (null)
2016-08-15 14:52:03.421 LKTestAutoreleasePool[36795:6319455] fraction_weak_@viewWillAppear: (null) with retainCount: (null)
2016-08-15 14:52:03.426 LKTestAutoreleasePool[36795:6319455] fraction_weak_@viewDidAppear: (null) with retainCount: (null)
```

### 场景3
```
__weak NSString *string_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 3.1
    NSString *string = [NSString stringWithFormat:@"I'm here!"];
    @autoreleasepool {
        NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
        string_weak_ = string;
        NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
        NSLog(@"string_weak_@viewDidLoad: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
    }

    NSLog(@"string@viewDidLoad: %@ with retainCount: %@", string, [string valueForKey:@"retainCount"]);
    NSLog(@"string_weak_@viewDidLoad: %@ with retainCount: %@", string_weak_, [string_weak_ valueForKey:@"retainCount"]);
}
```

```
__weak Fraction *fraction_weak_ = nil;

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景 3.2
    Fraction *fraction = [[Fraction alloc] init];
    @autoreleasepool {
        NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
        fraction_weak_ = fraction;
        NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
        NSLog(@"fraction_weak_@viewDidLoad: %@ with retainCount: %@", fraction_weak_, [fraction_weak_ valueForKey:@"retainCount"]);
    }

    NSLog(@"fraction@viewDidLoad: %@ with retainCount: %@", fraction, [fraction valueForKey:@"retainCount"]);
    NSLog(@"fraction_weak_@viewDidLoad: %@ with retainCount: %@", fraction_weak_, [fraction_weak_ valueForKey:@"retainCount"]);
}
```

```
// 场景3.1(__weak NSString *) 结果
2016-08-15 15:02:00.662 LKTestAutoreleasePool[36934:6331215] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 15:02:00.663 LKTestAutoreleasePool[36934:6331215] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 15:02:00.663 LKTestAutoreleasePool[36934:6331215] string_weak_@viewDidLoad: I'm here! with retainCount: 4
2016-08-15 15:02:00.663 LKTestAutoreleasePool[36934:6331215] string@viewDidLoad: I'm here! with retainCount: 2
2016-08-15 15:02:00.663 LKTestAutoreleasePool[36934:6331215] string_weak_@viewDidLoad: I'm here! with retainCount: 4
2016-08-15 15:02:00.664 LKTestAutoreleasePool[36934:6331215] string_weak_@viewWillAppear: I'm here! with retainCount: 3
2016-08-15 15:02:00.667 LKTestAutoreleasePool[36934:6331215] string_weak_@viewDidAppear: (null) with retainCount: (null)

// 场景3.2(__weak Fraction *) 结果
2016-08-15 14:54:43.872 LKTestAutoreleasePool[36821:6322969] fraction@viewDidLoad: <Fraction: 0x7ff031737ab0> with retainCount: 1
2016-08-15 14:54:43.873 LKTestAutoreleasePool[36821:6322969] fraction@viewDidLoad: <Fraction: 0x7ff031737ab0> with retainCount: 1
2016-08-15 14:54:43.873 LKTestAutoreleasePool[36821:6322969] fraction_weak_@viewDidLoad: <Fraction: 0x7ff031737ab0> with retainCount: 3
2016-08-15 14:54:43.873 LKTestAutoreleasePool[36821:6322969] fraction@viewDidLoad: <Fraction: 0x7ff031737ab0> with retainCount: 1
2016-08-15 14:54:43.873 LKTestAutoreleasePool[36821:6322969] fraction_weak_@viewDidLoad: <Fraction: 0x7ff031737ab0> with retainCount: 3
2016-08-15 14:54:43.874 LKTestAutoreleasePool[36821:6322969] fraction_weak_@viewWillAppear: (null) with retainCount: (null)
2016-08-15 14:54:43.878 LKTestAutoreleasePool[36821:6322969] fraction_weak_@viewDidAppear: (null) with retainCount: (null)
```

再来看@autoreleasepool之后的代码，fraction = [[Fraction alloc] init];
向Fraction类发送alloc消息，alloc方法我们并未定义，这来自于Fraction的父类NSObject的类方法，为一个Fraction对象分配了内存空间，并返回一个实例对象：
```
+ (instancetype)new OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)allocWithZone:(struct _NSZone *)zone OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)alloc OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
- (void)dealloc OBJC_SWIFT_UNAVAILABLE("use 'deinit' to define a de-initializer");
```

随后调用了init方法，同样，这来自于NSObject的实例方法。

fraction对象前的*表示fraction是Fraction对象的指针（引用），就是说fraction并不存储Fraction数据，而是保存了一个Fraction对象在内存中的地址。

而这个fraction对象在初始化时之时分配了空间，获得了他的内存地址，却没有设置值，因此我们setValue，print，得出了分数的值。
OC中调用方法可以理解为发送消息，因为在程序运行过程中所有OC代码都会被转化为runtime的C语言代码，`[target doSomething];会被转化成objc_msgSend(target, @selector(doSomething));`我们知道实例是对象，而类也是一个对象，而类在runtime中是一个结构体，会通过链表保存它的变量列表和方法列表等，在初始化时，也是根据结构体的数据结构为其分配内存。

提前阅读一些runtime的知识
[iOS~runtime理解](http://www.jianshu.com/p/927c8384855a)
[iOS中的runtime应用](http://www.jianshu.com/p/364eab29f4f5)
