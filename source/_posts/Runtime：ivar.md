---
title: Runtime：ivar
date: 2019-06-08 16:51:05
categories: Runtime
tags:
    - ivar
---
# ivar本质
```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */

struct objc_ivar_list {
    int ivar_count                                           OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;
```
`objc_class`中包含一个`objc_ivar_list`类型的`ivars`，管理了一个class中所有的成员变量，`objc_ivar_list`中包含`ivar_count`(ivar数量)、`space`(占用内存空间)、`ivar_list`(变长结构体)。
`objc_ivar`中包含着`ivar_name`(ivar名)、`ivar_type`(ivar类型)、`ivar_offset`(基地址偏移字节)、`space`(占用内存空间)。
# 变长结构体
要理解`ivar_list`要首先理解C语言中变长结构体的概念。
变长结构体最后一个元素是一个没有元素的数组，因此我们可以动态开辟一个比结构体更大的空间，让数组指针指向这块空间。
需要注意的是`ivar_list`与整个数据结构的内存地址是连续的，若是链表则是不连续的内存地址，连续内存有助于减少内存的碎片化，简化内存管理。
# 怎么添加ivar
我们常说OC是一门运行时语言，它有一部分东西是runtime时才决定的，但我们在常规开发中，很少去动态添加ivar。动态添加ivar主要有两种方式：
1. runtime动态关联对象。
2. `class_addIvar`，但是它只能在`objc_allocateClassPair`和`objc_allocateClassPair`两个函数之间调用，因此只有在runtime中动态创建class时才能动态添加ivar。
## 关联对象原理
其实关联对象的原理与我们在分类中添加一个全局变量的get\set方法类似，关联对象并没有直接加在class中，而是添加在`AssociationsManager`的hash map里与class关联了起来。
可以参考[iOS底层原理总结 - 关联对象实现原理](https://blog.csdn.net/olsq93038o99s/article/details/80878983)