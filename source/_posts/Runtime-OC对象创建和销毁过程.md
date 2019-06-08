---
title: Runtime-OC对象创建和销毁过程
date: 2019-06-08 15:59:08
categories: Runtime
tags:
    - OC对象
---
# 对象创建
OC的创建对象，如`[[MyClass alloc] init]`，从消息发送角度说，就是给MyClass类对象发了两个消息，逐层lookup找到NSObject中的这两个方法，随后根据MyClass所需内存空间大小分配内存，然后把isa指针指向MyClass，最后将这两个方法加入缓存方法列表。
那么再往底层看，Runtime具体做了些什么呢，
## + (id)alloc
```
+ (id)alloc {
    return _objc_rootAlloc(self);
}

id _objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    // 做类型检查
    if (checkNil && !cls) return nil;

#if __OBJC2__
    // 若没有实现allocWithZone方法
    if (! cls->ISA()->hasCustomAWZ()) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        // canAllocFast默认返回false，应该是有一些类做了系统优化直接放到bits里了
        if (cls->canAllocFast()) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (!obj) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        } else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (!obj) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```
可以看到我们最关心最常用的是`class_createInstance`方法，接下来看看这里面干了些啥。
```
id class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}

static __attribute__((always_inline)) id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocIndexed();

    // 字节对齐，OC对象16字节对齐，一个对象最少16字节，isa 8字节。一个对象的内存地址从其第一个成员开始，也就是从isa指针开始。
    size_t size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    // 分配内存空间，创建对象
    id obj;
    if (!UseGC  &&  !zone  &&  fast) {
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
#if SUPPORT_GC
        if (UseGC) {
            obj = (id)auto_zone_allocate_object(gc_zone, size,
                                                AUTO_OBJECT_SCANNED, 0, 1);
        } else 
#endif
        if (zone) {
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else {
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        // Use non-indexed isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls);
    }

    return obj;
}
```
## + (id)init
```
// Replaced by CF (throws an NSException)
+ (id)init {
    return (id)self;
}

- (id)init {
    return _objc_rootInit(self);
}

id _objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```
苹果并没有做什么事情，让人觉得其实只需要写`[MyClass alloc]`就好了，不过你要是真写出如下代码，是会抛出`EXC_BAD_ACCESS (code=1, address=0x0)`Crash的。
```
self.testView = [UIView alloc];
self.testView.frame = CGRectMake(50, 50, 100, 100);
self.testView.backgroundColor = [UIColor blueColor];
[self.view addSubview:self.testView];
```

# 对象销毁
```
// Replaced by NSZombies
// 应该是与debug时的僵尸对象有关，苹果可能是通过类似方法交换的黑魔法对dealloc做了一些事情。
- (void)dealloc {
    _objc_rootDealloc(self);
}

void _objc_rootDealloc(id obj) {
    assert(obj);

    obj->rootDealloc();
}

inline void objc_object::rootDealloc() {
    assert(!UseGC);
    if (isTaggedPointer()) return;

    // 如果没有weak引用 && 没有关联对象 && 没有c++析构 && 没有side table借位
    // 释放内存空间
    if (isa.indexed  &&  
        !isa.weakly_referenced  &&  
        !isa.has_assoc  &&  
        !isa.has_cxx_dtor  &&  
        !isa.has_sidetable_rc) {
        assert(!sidetable_present());
        free(this);
    } else {
        object_dispose((id)this);
    }
}
```
再继续看object_dispose方法
```
id object_dispose(id obj) {
    if (!obj) return nil;

    objc_destructInstance(obj);
    
#if SUPPORT_GC
    if (UseGC) {
        auto_zone_retain(gc_zone, obj); // gc free expects rc==1
    }
#endif

    free(obj);

    return nil;
}

/***********************************************************************
* objc_destructInstance
* Destroys an instance without freeing memory. 
* Calls C++ destructors.
* Calls ARR ivar cleanup.
* Removes associative references.
* Returns `obj`. Does nothing if `obj` is nil.
* Be warned that GC DOES NOT CALL THIS. If you edit this, also edit finalize.
* CoreFoundation and other clients do call this under GC.
**********************************************************************/
void *objc_destructInstance(id obj) {
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = !UseGC && obj->hasAssociatedObjects();
        bool dealloc = !UseGC;

        // This order is important.
        // 特别强调了这里的时序很重要
        if (cxx) object_cxxDestruct(obj);               // C++析构函数
        if (assoc) _object_remove_assocations(obj);     // 移除关联对象
        if (dealloc) obj->clearDeallocating();          // 清理引用
    }

    return obj;
}

inline void objc_object::clearDeallocating() {
    if (!isa.indexed) {
        // Slow path for raw pointer isa.
        // 清理sideTable中的引用计数表
        sidetable_clearDeallocating();
    } else if (isa.weakly_referenced  ||  isa.has_sidetable_rc) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        // 清理sideTable中的weak表
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}
```