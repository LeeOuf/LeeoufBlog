---
title: 开源库阅读笔记：YYKit-YYModel
date: 2017-03-13 18:33:05
categories: iOS
tags: 
    - 开源库阅读笔记
---

YYKit，作者ibireme，[Blog](http://blog.ibireme.com)，[YYKitDemo](https://github.com/ibireme/YYKit)。

# YYModel
YYModel作为NSObject的Category，具有JSON字符串与Model相互转化的功能。
JSON解析过程如下：
1. 判空，获取class。
2. 若用户自定义class，则根据元类YYModelMeta获取class（如根据字段将类分为圆、正方形等）。
3. 根据JSON转化Model。
自定义class属于扩展功能，先分析JSON转化Model这一主要功能。
<!-- more -->

## JSON转化Model
此功能核心方法如下：
```
- (BOOL)modelSetWithDictionary:(NSDictionary *)dic {
    if (!dic || dic == (id)kCFNull) return NO;
    if (![dic isKindOfClass:[NSDictionary class]]) return NO;
    
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:object_getClass(self)];
    if (modelMeta->_keyMappedCount == 0) return NO;
    
    if (modelMeta->_hasCustomWillTransformFromDictionary) {
        dic = [((id<YYModel>)self) modelCustomWillTransformFromDictionary:dic];
        if (![dic isKindOfClass:[NSDictionary class]]) return NO;
    }
    
    ModelSetContext context = {0};
    context.modelMeta = (__bridge void *)(modelMeta);
    context.model = (__bridge void *)(self);
    context.dictionary = (__bridge void *)(dic);
    
    if (modelMeta->_keyMappedCount >= CFDictionaryGetCount((CFDictionaryRef)dic)) {
        CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);
        if (modelMeta->_keyPathPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_keyPathPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_keyPathPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
        if (modelMeta->_multiKeysPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_multiKeysPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_multiKeysPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
    } else {
        CFArrayApplyFunction((CFArrayRef)modelMeta->_allPropertyMetas,
                             CFRangeMake(0, modelMeta->_keyMappedCount),
                             ModelSetWithPropertyMetaArrayFunction,
                             &context);
    }
    
    if (modelMeta->_hasCustomTransformFromDictionary) {
        return [((id<YYModel>)self) modelCustomTransformFromDictionary:dic];
    }
    return YES;
}
```
其思路简单明了，通过class获取YYMoel元类，随后根据YYModelMeta递归调用自身，直到将Model中所有属性转化完成，接下来仔细阅读代码。

### 异常处理
首先对传入的NSDictionary进行了判空处理，如下。
```
if (!dic || dic == (id)kCFNull) return NO;
if (![dic isKindOfClass:[NSDictionary class]]) return NO;
```
> 在开发时对NSDictionary的判空一般使用nil，kCFNull是什么呢？
> iOS中的空值类型有5种：nil\Nil\NULL\NSNull\kCFNull。
> nil：指向一个实例对象的空指针。
> Nil：指向一个类的空指针。
> NULL：定义基本类型、C类型的空指针。
> NSNull：数组中元素的占位符，数组元素若为nil则数组结束，但是可以为NSNull。
> kCFNull：NSNull的单例。
> 参考：[IOS 空值 nil Nil NULL NSNull kCFNull](http://www.jianshu.com/p/3aaefb3bcf73)

### 获取元类
```
_YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:object_getClass(self)];
```
要想理解这部分思想，首先需要理解Objective-C对象和runtime的概念。
> ![对象模型图](/对象模型图.png)
> 可参考：[Objective-C中的类和对象](http://blog.ibireme.com/2013/11/25/objc-object/)
> [object_getClass(obj)与[obj class]的区别](http://www.jianshu.com/p/ae5c32708bc6)
> 测试代码如下：
```
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    TestClass *obj1 = [[TestClass alloc] init];
    TestClass *obj2 = [[TestClass alloc] init];
    
    Class cls1 = object_getClass(obj1);
    Class cls2 = object_getClass(cls1);
    Class cls3 = object_getClass(cls2);
    NSLog(@"%@, %p", obj1, obj1);
    NSLog(@"%@, %p", cls1, cls1);
    NSLog(@"%@, %p", cls2, cls2);
    NSLog(@"%@, %p", cls3, cls3);
    
    Class cls4 = [obj1 class];
    Class cls5 = [cls4 class];
    Class cls6 = [cls5 class];
    NSLog(@"%@, %p", obj2, obj2);
    NSLog(@"%@, %p", cls4, cls4);
    NSLog(@"%@, %p", cls5, cls5);
    NSLog(@"%@, %p", cls6, cls6);
    
    Class cls7 = [cls1 class];
    Class cls8 = [cls2 class];
    Class cls9 = [cls3 class];
    NSLog(@"%@, %p", cls7, cls7);
    NSLog(@"%@, %p", cls8, cls8);
    NSLog(@"%@, %p", cls9, cls9);
}
```
> 
> 其中TestClass继承TestSuperClass，TestSuperClass继承NSObject。
> 结果如下：
```
2017-04-14 12:41:23.561943+0800 LKTestApp[829:229808] <TestClass: 0x174007f90>, 0x174007f90
2017-04-14 12:41:23.562018+0800 LKTestApp[829:229808] TestClass, 0x100101280
2017-04-14 12:41:23.562037+0800 LKTestApp[829:229808] TestClass, 0x100101258
2017-04-14 12:41:23.562051+0800 LKTestApp[829:229808] NSObject, 0x1a9129ec8
2017-04-14 12:41:23.562090+0800 LKTestApp[829:229808] <TestClass: 0x174007fa0>, 0x174007fa0
2017-04-14 12:41:23.562103+0800 LKTestApp[829:229808] TestClass, 0x100101280
2017-04-14 12:41:23.562115+0800 LKTestApp[829:229808] TestClass, 0x100101280
2017-04-14 12:41:23.562126+0800 LKTestApp[829:229808] TestClass, 0x100101280
2017-04-14 12:41:23.562138+0800 LKTestApp[829:229808] TestClass, 0x100101280
2017-04-14 12:41:23.562149+0800 LKTestApp[829:229808] TestClass, 0x100101258
2017-04-14 12:41:23.562160+0800 LKTestApp[829:229808] NSObject, 0x1a9129ec8
```
> 
> 可知object_getClass(obj)返回了obj的isa指针，[obj class]obj为实例对象时调用实例方法，返回isa指针，若为类对象则调用类方法返回自身。

然后我们来看看YYModel中的元类`_YYModelMeta`，其数据结构如下：
```
@interface _YYModelMeta : NSObject {
    @package
    YYClassInfo *_classInfo;
    /// Key:mapped key and key path, Value:_YYModelPropertyMeta.
    NSDictionary *_mapper;
    /// Array<_YYModelPropertyMeta>, all property meta of this model.
    NSArray *_allPropertyMetas;
    /// Array<_YYModelPropertyMeta>, property meta which is mapped to a key path.
    NSArray *_keyPathPropertyMetas;
    /// Array<_YYModelPropertyMeta>, property meta which is mapped to multi keys.
    NSArray *_multiKeysPropertyMetas;
    /// The number of mapped key (and key path), same to _mapper.count.
    NSUInteger _keyMappedCount;
    /// Model class type.
    YYEncodingNSType _nsType;
    
    BOOL _hasCustomWillTransformFromDictionary;
    BOOL _hasCustomTransformFromDictionary;
    BOOL _hasCustomTransformToDictionary;
    BOOL _hasCustomClassFromDictionary;
}
@end
```
> Objective-C中有4种访问控制符：
> @private（当前类访问权限）：成员只能在当前类内部可以访问，在类实现部分定义的成员变量相当于默认使用了这种访问权限。
> @package（同映像访问权限）：成员可以在当前类或和当前类实现的同一映像中使用。同一映像就是编译后生成的同一框架或同一个执行文件。
> @protected（子类访问权限）：成员可以在当前类和当前类的子类中访问。在类接口部分定义的成员变量默认是这种访问权限。
> @public（公共访问权限）：成员可以在任意地方访问。

`_YYModelMeta`存储了原始key value的映射信息，将多个key对应一个value(keyPath)和一个key对应多个value(keyArray)等复杂情况交给`_YYModelPropertyMeta`处理。
```
@interface _YYModelPropertyMeta : NSObject {
    @package
    NSString *_name;             ///< property's name
    YYEncodingType _type;        ///< property's type
    YYEncodingNSType _nsType;    ///< property's Foundation type
    BOOL _isCNumber;             ///< is c number type
    Class _cls;                  ///< property's class, or nil
    Class _genericCls;           ///< container's generic class, or nil if threr's no generic class
    SEL _getter;                 ///< getter, or nil if the instances cannot respond
    SEL _setter;                 ///< setter, or nil if the instances cannot respond
    BOOL _isKVCCompatible;       ///< YES if it can access with key-value coding
    BOOL _isStructAvailableForKeyedArchiver; ///< YES if the struct can encoded with keyed archiver/unarchiver
    BOOL _hasCustomClassFromDictionary; ///< class/generic class implements +modelCustomClassForDictionary:
    
    /*
     property->key:       _mappedToKey:key     _mappedToKeyPath:nil            _mappedToKeyArray:nil
     property->keyPath:   _mappedToKey:keyPath _mappedToKeyPath:keyPath(array) _mappedToKeyArray:nil
     property->keys:      _mappedToKey:keys[0] _mappedToKeyPath:nil/keyPath    _mappedToKeyArray:keys(array)
     */
    NSString *_mappedToKey;      ///< the key mapped to
    NSArray *_mappedToKeyPath;   ///< the key path mapped to (nil if the name is not key path)
    NSArray *_mappedToKeyArray;  ///< the key(NSString) or keyPath(NSArray) array (nil if not mapped to multiple keys)
    YYClassPropertyInfo *_info;  ///< property's info
    _YYModelPropertyMeta *_next; ///< next meta if there are multiple properties mapped to the same key.
}
@end

@interface YYClassPropertyInfo : NSObject
@property (nonatomic, assign, readonly) objc_property_t property; ///< property's opaque struct
@property (nonatomic, strong, readonly) NSString *name;           ///< property's name
@property (nonatomic, assign, readonly) YYEncodingType type;      ///< property's type
@property (nonatomic, strong, readonly) NSString *typeEncoding;   ///< property's encoding value
@property (nonatomic, strong, readonly) NSString *ivarName;       ///< property's ivar name
@property (nullable, nonatomic, assign, readonly) Class cls;      ///< may be nil
@property (nullable, nonatomic, strong, readonly) NSArray<NSString *> *protocols; ///< may nil
@property (nonatomic, assign, readonly) SEL getter;               ///< getter (nonnull)
@property (nonatomic, assign, readonly) SEL setter;               ///< setter (nonnull)
@end
```

`YYClassInfo`则与OC中类的结构相似，包含了父类指针、元类指针、是否是元类以及属性、方法、ivar列表等属性。
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
@end
```
> ivar是objc_ivar类型的实例变量指针，而属性则是objc_property。

随后通过`+ (instancetype)metaWithClass:(Class)cls;`进行元类初始化。此方法中主要涉及内存缓存和锁，涉及的知识点如下：
> 1. [CFMutableDictionaryRef](https://developer.apple.com/reference/corefoundation/cfmutabledictionary-rpl)是Core Foundation中的类，此处使用它而不使用NSMuatableDictionary可能是因为底层类库的需要
> 2. dispatch_semaphore信号量，包括dispatch_semaphore_create，dispatch_semaphore_wait，dispatch_semaphore_signal等用法，可查看[资料1](http://www.jianshu.com/p/a84c2bf0d77b)[资料2](http://blog.ibireme.com/2015/10/26/yycache/)
> 3. 两个时间宏
> #define DISPATCH_TIME_NOW (0ull)
> #define DISPATCH_TIME_FOREVER (~0ull)
> 0ull是unsigned long long类型的0，~则表示二进制否运算，因此~0ull是0ull取非，因此是2^64，一个无符号的极大数

真正的初始化方法是`- (instancetype)initWithClass:(Class)cls;`，其流程如下：
1. 获取YYClassInfo。
2. 获取黑名单。
3. 获取白名单。
4. 获取用户自定义属性。
5. 循环获取所有属性，生成_YYModelPropertyMeta实例，将其存入`_allPropertyMetas`、`_keyPathPropertyMetas`、`_multiKeysPropertyMetas`。
6. 设置属性映射mapper。

至此已获取YYModel元类，包括key/value属性映射关系，回到`- (BOOL)modelSetWithDictionary:(NSDictionary *)dic;`进行JSON转化。如下，根据_YYModelMeta对json进行递归，将value依次设置到model属性中。
```
ModelSetContext context = {0};
context.modelMeta = (__bridge void *)(modelMeta);
context.model = (__bridge void *)(self);
context.dictionary = (__bridge void *)(dic);

if (modelMeta->_keyMappedCount >= CFDictionaryGetCount((CFDictionaryRef)dic)) {
    CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);
    if (modelMeta->_keyPathPropertyMetas) {
        CFArrayApplyFunction((CFArrayRef)modelMeta->_keyPathPropertyMetas,
                             CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_keyPathPropertyMetas)),
                             ModelSetWithPropertyMetaArrayFunction,
                             &context);
    }
    if (modelMeta->_multiKeysPropertyMetas) {
        CFArrayApplyFunction((CFArrayRef)modelMeta->_multiKeysPropertyMetas,
                             CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_multiKeysPropertyMetas)),
                             ModelSetWithPropertyMetaArrayFunction,
                             &context);
    }
} else {
    CFArrayApplyFunction((CFArrayRef)modelMeta->_allPropertyMetas,
                         CFRangeMake(0, modelMeta->_keyMappedCount),
                         ModelSetWithPropertyMetaArrayFunction,
                         &context);
}

typedef struct {
    void *modelMeta;  ///< _YYModelMeta
    void *model;      ///< id (self)
    void *dictionary; ///< NSDictionary (json)
} ModelSetContext;
```
> SEL即selector，可通过@selector或NSSelectorFromString创建，是一种方法选择器，描述了方法的格式，但并不真正指向方法。
> IMP即implement，指向函数，包含接收消息对象、SEL、入参、返回值等。
> 
> KVC(key value coding)
> KVC涉及以下4个方法
```
- (nullable id)valueForKey:(NSString *)key;                          //直接通过Key来取值
- (void)setValue:(nullable id)value forKey:(NSString *)key;          //通过Key来设值
- (nullable id)valueForKeyPath:(NSString *)keyPath;                  //通过KeyPath来取值
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;  //通过KeyPath来设值
```
> [iOS开发技巧系列---详解KVC](http://www.jianshu.com/p/45cbd324ea65)
> 
> KVO(key value obeserveing)
> KVO涉及以下2种方法，addObserver和removeObserver，用于监听对象属性的变化。
```
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
```
> [IOS 开发中的KVC 和KVO](http://www.jianshu.com/p/c1dd046d9ce2)

扩展阅读：
[看 YYModel 源码的一些收获](http://www.cocoachina.com/cms/wap.php?action=article&id=17874)
> 性能优化Tip
> 缓存Model JSON 转换过程中需要很多类的元数据，如果数据足够小，则全部缓存到内存中。
> 查表当遇到多项选择的条件时，要尽量使用查表法实现，比如 switch/case，C Array，如果查表条件是对象，则可以用 NSDictionary 来实现。
> 避免 KVCKey-Value Coding 使用起来非常方便，但性能上要差于直接调用 Getter/Setter，所以如果能避免 KVC 而用 Getter/Setter 代替，性能会有较大提升。
> 避免 Getter/Setter 调用如果能直接访问 ivar，则尽量使用 ivar 而不要使用 Getter/Setter 这样也能节省一部分开销。
> 避免多余的内存管理方法在 ARC 条件下，默认声明的对象是 strong 类型的，赋值时有可能会产生 retain/release 调用，如果一个变量在其生命周期内不会被释放，则使用 unsafe_unretained 会节省很大的开销。访问具有 weak 属性的变量时，实际上会调用 objc_loadWeak() 和 objc_storeWeak() 来完成，这也会带来很大的开销，所以要避免使用 weak 属性。创建和使用对象时，要尽量避免对象进入 autoreleasepool，以避免额外的资源开销。
> 遍历容器类时，选择更高效的方法相对于 Foundation 的方法来说，CoreFoundation 的方法有更高的性能，用 CFArrayApplyFunction() 和 CFDictionaryApplyFunction() 方法来遍历容器类能带来不少性能提升，但代码写起来会非常麻烦。
> 尽量用纯 C 函数、内联函数使用纯 C 函数可以避免 ObjC 的消息发送带来的开销。如果 C 函数比较小，使用 inline 可以避免一部分压栈弹栈等函数调用的开销。
> 减少遍历的循环次数在 JSON 和 Model 转换前，Model 的属性个数和 JSON 的属性个数都是已知的，这时选择数量较少的那一方进行遍历，会节省很多时间。