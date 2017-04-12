---
title: 开源库阅读笔记：YYKit
date: 2017-03-13 18:33:05
categories: iOS
tags: 
    - 开源库阅读笔记
---

YYKit，作者ibireme，[Blog](http://blog.ibireme.com)，[YYKitDemo](https://github.com/ibireme/YYKit)。

# 印象
YYKitDemo分为Model/Image/Text/Feed List Demo四个部分，代码大部分为MRC，极少部分ARC。

# YYRootViewController
从`YYRootVC`看起，这是一个很普通的UITableViewController子类，不过是rootViewController。其中包含了VC的生命周期、UITableViewDataSource和一个log方法。
类中有两个NSMutableArray的私有实例变量，作为UITableView的数据源，唯一特殊的一点是作者使用了一种比较花哨的初始化方式`self.titles = @[].mutableCopy;`，与普通的init并没有区别。
log方法是一个递归方法，用于打印内存，内存容量统计写于`UIDevice+YYAdd`中，可以作为标准写法来使用。
<!-- more -->

# YYModelExample
这个demo展示了YYModel的用法，YYModel的主要功能是`modelWithJSON`和`modelToJSONString`等json字符串与model转换的方法，支持嵌套自定义类、NSArray、NSDictionary、NSSet、自定义类数组、自定义映射以及coding/copying/hash/equal等扩展功能。
YYModel分为4个部分，NSObject(YYModel)、NSArray(YYModel)、NSDictionary(YYModel)、YYModel protocol。

## modelWithJSON方法
从modelWithJSON方法作为入口从上往下看，该方法在NSObject(YYModel)中，返回值为instancetype，入参为id类型，方便在任何地方灵活使用。
这个方法首先将id类型入参转化为NSDictionary，转换方法为_yy_dictionaryWithJSON，若入参为NSDictionary则直接返回，若为NSString或NSData则转化为NSDictionary，其他情况返回nil。
> NSString转化NSData类型使用NSString - dataUsingEncoding:，NSData转化NSDictionary使用NSJSONSerialization + JSONObjectWithData:

成功转化后调用modelWithDictionary方法。

## modelWithDictionary
经过一段异常处理判空逻辑后进入正题。
1. [self class]获取类，这也是YYModel作为NSObject Category的原因。
2. [_YYModelMeta metaWithClass:cls]
创建元类，首先这里使用了内存缓存，通过单例建立缓存，缓存过程中使用GCD dispatch_semaphore信号量加锁，将创建的所有元类存入可变数组，若缓存中已有该元类并且无需更新，则直接取出并返回，否则init一个。
> 1. [CFMutableDictionaryRef](https://developer.apple.com/reference/corefoundation/cfmutabledictionary-rpl)是Core Foundation中的类，此处使用它而不使用NSMuatableDictionary可能是因为底层类库的需要
> 2. dispatch_semaphore信号量，包括dispatch_semaphore_create，dispatch_semaphore_wait，dispatch_semaphore_signal等用法，可查看[资料1](http://www.jianshu.com/p/a84c2bf0d77b)[资料2](http://blog.ibireme.com/2015/10/26/yycache/)
> 3. 两个时间宏
> #define DISPATCH_TIME_NOW (0ull)
> #define DISPATCH_TIME_FOREVER (~0ull)
> 0ull是unsigned long long类型的0，~则表示二进制否运算，因此~0ull是0ull取非，因此是2^64，一个无符号的极大数
> 4. 访问控制符
> OC中提供了4个访问控制符：@private @package @protected @public。
> @private（当前类访问权限）：成员只能在当前类内部可以访问，在类实现部分定义的成员变量相当于默认使用了这种访问权限。
> @package（同映像访问权限）：成员可以在当前类或和当前类实现的同一映像中使用。同一映像就是编译后生成的同一框架或同一个执行文件。
> @protected（子类访问权限）：成员可以在当前类和当前类的子类中访问。在类接口部分定义的成员变量默认是这种访问权限。
> @public（公共访问权限）：成员可以在任意地方访问。

## [[_YYModelMeta alloc] initWithClass:cls]
该方法返回的是YYModelMeta本身，NSObject子类，即model元类。这里有一个新类YYClassInfo，以下为该方法变量及方法。
```
@interface YYClassInfo : NSObject
@property (nonatomic, assign, readonly) Class cls; ///< class object
@property (nullable, nonatomic, assign, readonly) Class superCls; ///< super class object
@property (nullable, nonatomic, assign, readonly) Class metaCls;  ///< class's meta class object
@property (nonatomic, readonly) BOOL isMeta; ///< whether this class is meta class
@property (nonatomic, strong, readonly) NSString *name; ///< class name
@property (nullable, nonatomic, strong, readonly) YYClassInfo *superClassInfo; ///< super class's class info
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassIvarInfo *> *ivarInfos; ///< ivars
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassMethodInfo *> *methodInfos; ///< methods
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassPropertyInfo *> *propertyInfos; ///< properties

- (void)setNeedUpdate;
- (BOOL)needUpdate;
+ (nullable instancetype)classInfoWithClass:(Class)cls;
+ (nullable instancetype)classInfoWithClassName:(NSString *)className;
```
YYModelMeta的结构与OC中对象的定义类似，可参考[Objective-C中的类和对象](http://blog.ibireme.com/2013/11/25/objc-object/)











