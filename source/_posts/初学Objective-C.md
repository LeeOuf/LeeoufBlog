---
title: 初学Objective-C
date: 2016-08-16 15:36:56
categories: iOS基础知识
tags:
    - Objective-C
    - 类
    - 属性
    - 方法
    - 惰性实例化
---
我们知道iOS中的frame由x, y, width, height决定的一个个矩形。
接下来尝试用一个矩形类来探究如何使用类中的属性和方法。
<!-- more -->

**坐标类XYPoint。**
```
#import <Foundation/Foundation.h>

@interface XYPoint : NSObject

@property int x, y;

- (void)setX:(int)xVal andY:(int)yVal;

@end
```

```
#import "XYPoint.h"

@implementation XYPoint

-(void)setX:(int)xVal andY:(int)yVal {
    _x = xVal;
    _y = yVal;
}

@end
```
<!-- more -->

**矩形类MyRect**
```
#import <Foundation/Foundation.h>
#import "XYPoint.h"

@interface MyRect : NSObject

@property (nonatomic) int width, height;
@property (nonatomic) XYPoint *origin;

- (void)setWidth:(int)w andHeight:(int)h;
- (int)area;
- (int)perimeter;

@end
```

```
#import "MyRect.h"

@implementation MyRect

- (XYPoint *)origin {
    return _origin;
}

- (void)setOrigin:(XYPoint *)origin {
    _origin = origin;
}

- (void)setWidth:(int)w andHeight:(int)h {
    _width = w;
    _height = h;
}

- (int)area {
    return _width * _height;
}

- (int)perimeter {
    return (_width + _height) * 2;
}

@end
```

**main函数**
```
#import <Foundation/Foundation.h>
#import "MyRect.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyRect *rect = [[MyRect alloc] init];
        XYPoint *point = [[XYPoint alloc] init];
        
        [point setX:100 andY:200];
        
        [rect setWidth:5 andHeight:8];
        rect.origin = point;
        
        NSLog(@"Rect w = %i, h = %i", rect.width, rect.height);
        NSLog(@"Origin at (%i, %i)", rect.origin.x, rect.origin.y);
        
        [point setX:50 andY:50];
        NSLog(@"Origin at (%i, %i)", rect.origin.x, rect.origin.y);
//        NSLog(@"Area = %i, Perimeter = %i", [rect area], [rect perimeter]);
    }
    return 0;
}
```

XYPoint类非常简单，x，y属性，加上其set方法。
MyRect类中有width，height以及一个XYPoint属性，加上一些方法。
main函数中是我们编写的测试代码，做一些初始化，赋值，输出的操作。

我们的重点在于MyRect类，这段代码是有问题的。
首先看头文件，利用@property自动声明get\set方法。
再看实现文件，在get方法中报`Use of undeclared indentifier '_origin'`error。说好的在Xcode4.5以后可以省略@synthesize呢？现在我们删除get方法和set方法其中之一，发现error消失了。实际上编译器确实自动添加了@synthesize，默认的实例变量名为`"_" + 属性名`。但是当我们同时重写了get\set方法时，系统就不再自动帮我生成实例变量了。

现在我们加上`@synthesize origin = _origin;`。error彻底消失，我们也可以自由地重写get\set方法了。
运行main函数得到结果。
```
2016-08-16 15:59:45.763 LKTestPracticeClass[39983:6991728] Rect w = 5, h = 8
2016-08-16 15:59:45.764 LKTestPracticeClass[39983:6991728] Origin at (100, 200)
2016-08-16 15:59:45.764 LKTestPracticeClass[39983:6991728] Origin at (50, 50)
Program ended with exit code: 0
```

回到main函数，我们将point对象赋值给rect的属性origin，然后改变point对象的值，为什么rect的origin属性值也随之改变了呢？我们知道C语言中函数的参数都是值传递，那么来看MyRect类中origin的set方法，
```
- (void)setOrigin:(XYPoint *)origin {
    _origin = origin;
}
```
origin是(XYPoint *)类型，这是一个XYPoint类型对象的引用/地址，而`_origin = origin;`只是将这个对象的地址赋值给了实例变量`_origin`，现在，它们指向了同一个内存地址，那么我们改变了这个地址所指向的值，则point对象和rect对象的origin对象值都同时改变了。

如何解决这个问题呢，这里有一个重点的思想，rect对象应该持有一个XYPoint类型的对象，这是它的横纵坐标，这个对象应该有它自己的内存空间，而不是指向一块与rect对象无关的内存。

第一个方案，改造setOrigin方法。
```
- (void)setOrigin:(XYPoint *)origin {
    if (!_origin) {
        _origin = [[XYPoint alloc] init];
    }
    _origin.x = origin.x;
    _origin.y = origin.y;
}
```
当我们给origin set值时，实例化一块内存空间，然后将传递过来的point对象中x，y一一赋值给新的origin对象。
运行结果：
```
2016-08-16 16:09:37.557 LKTestPracticeClass[40037:6999091] Rect w = 5, h = 8
2016-08-16 16:09:37.558 LKTestPracticeClass[40037:6999091] Origin at (100, 200)
2016-08-16 16:09:37.558 LKTestPracticeClass[40037:6999091] Origin at (100, 200)
Program ended with exit code: 0
```

很明显，这生效了，但是在set方法中实例化这让人难受，更舒服的做法是每当我获取origin属性的时候，再做实例化，因为我可能并不会重写set方法，或许我不止这一个set方法，总之set方法太“上游”了，我希望在更底层、更统一的地方做init操作。因此，我们把实例化的代码移动到get中，这也叫做惰性实例化。
```
- (XYPoint *)origin {
    if (!_origin) {
        _origin = [[XYPoint alloc] init];
    }
    return _origin;
}

- (void)setOrigin:(XYPoint *)origin {
    _origin.x = origin.x;
    _origin.y = origin.y;
}
```
运行代码，期待...
```
2016-08-16 16:14:26.337 LKTestPracticeClass[40065:7002556] Rect w = 5, h = 8
2016-08-16 16:14:26.338 LKTestPracticeClass[40065:7002556] Origin at (0, 0)
2016-08-16 16:14:26.338 LKTestPracticeClass[40065:7002556] Origin at (0, 0)
Program ended with exit code: 0
```
然而结果是这样的，x，y的值从始至终都是0。

我们检查一下main函数，原来我们从未调用过get方法，origin属性从未指向任何内存，也就是nil，而nil对象中值是从0开始的。那么，让我们在setOrigin之前调用一个get吧。
```
rect.origin;
rect.origin = point;
```
运行，
```
2016-08-16 16:17:18.664 LKTestPracticeClass[40097:7005323] Rect w = 5, h = 8
2016-08-16 16:17:18.664 LKTestPracticeClass[40097:7005323] Origin at (100, 200)
2016-08-16 16:17:18.664 LKTestPracticeClass[40097:7005323] Origin at (100, 200)
Program ended with exit code: 0
```
很棒，实现了我们的目的。

可是这个get方法调用得莫名其妙，异常丑陋，让我们删掉它，再研究一下MyRect类。我们在set方法中调用一下get不就实现了我们的目的吗？
于是...
```
@synthesize origin = _origin;

- (XYPoint *)origin {
    if (!_origin) {
        _origin = [[XYPoint alloc] init];
    }
    return _origin;
}

- (void)setOrigin:(XYPoint *)origin {
    self.origin.x = origin.x;
    self.origin.y = origin.y;
}
```
这里的self.origin就是origin的get方法（我们只在获取property时用点操作符，其他时候仍然用方括号）。

最后，整理一下思路，`MyRect`类中有一个`origin`属性，我们在它的实现中为它指定了实例变量`_origin`作为它的存储空间。然后我们重写了它的get\set方法，在get方法中，我们采取惰性实例化，没有重写init方法在一开始就为实例变量开辟空间，而是直到我们不得不去获取它的值时，才进行初始化。接下来，在其他的方法中，我们一律用get方法获取`_origin`实例变量，它是实例变量的入口，不管我们想要获取它的值，还是改变它的值，都能够确保它开辟了一块内存空间用于存储，而且是在最后一刻万不得已时才初始化的，在不必要的时候没有浪费一丝内存。
另一方面，对于int类型的width和height来说，则没有惰性实例化这一概念，因为它们是数字字符常量，处于常量区中，一直都持有着属于它们的内存空间。

来看一下改造完的代码。
MyRect类实现。
```
#import "MyRect.h"

@implementation MyRect

@synthesize origin = _origin;

- (XYPoint *)origin {
    if (!_origin) {
        _origin = [[XYPoint alloc] init];
    }
    return _origin;
}

- (void)setOrigin:(XYPoint *)origin {
    self.origin.x = origin.x;
    self.origin.y = origin.y;
}

- (void)setWidth:(int)w andHeight:(int)h {
    self.width = w;
    self.height = h;
}

- (int)area {
    return self.width * self.height;
}

- (int)perimeter {
    return (self.width + self.height) * 2;
}

@end
```

main函数
```
#import <Foundation/Foundation.h>
#import "MyRect.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyRect *rect = [[MyRect alloc] init];
        XYPoint *point = [[XYPoint alloc] init];
        
        [point setX:100 andY:200];
        
        [rect setWidth:5 andHeight:8];
        rect.origin = point;
        
        NSLog(@"Rect w = %i, h = %i", rect.width, rect.height);
        NSLog(@"Origin at (%i, %i)", rect.origin.x, rect.origin.y);
        
        [point setX:50 andY:50];
        NSLog(@"Origin at (%i, %i)", rect.origin.x, rect.origin.y);
        NSLog(@"Area = %i, Perimeter = %i", [rect area], [rect perimeter]);
    }
    return 0;
}
```

运行结果
```
2016-08-16 16:44:37.009 LKTestPracticeClass[40218:7027614] Rect w = 5, h = 8
2016-08-16 16:44:37.010 LKTestPracticeClass[40218:7027614] Origin at (100, 200)
2016-08-16 16:44:37.010 LKTestPracticeClass[40218:7027614] Origin at (100, 200)
2016-08-16 16:44:37.010 LKTestPracticeClass[40218:7027614] Area = 40, Perimeter = 26
Program ended with exit code: 0
```