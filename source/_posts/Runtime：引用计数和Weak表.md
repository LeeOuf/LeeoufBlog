---
title: Runtime：引用计数和Weak表
date: 2019-06-07 14:52:34
categories: Runtime
tags:
    - 引用计数
    - Weak表
---
retainCount(引用计数)可以说是iOS内存管理的基础，当一个对象的retianCount为0时，没有任何地方引用，该对象的内存就会被释放。iOS进入ARC时代后RD们已经很少看和写`retain release`这些操作引用计数的代码，那么我们进入Runtime看看底层都做了些什么。
# SideTables
```
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```
在iOS中StripedMap管理着一个长度64(StripeCount)的数组，其本身是一个`void * -> T`的map，通过`indexForPointer`方法找到对应的数组下标，在hash过程中通过取余StripeCount来防止数组越界。
StripedMap的作用不仅限于管理SideTable，还可以用来管理spinlock_t(自旋锁)或其他包含着自旋锁的数据结构。

# SideTable
```
struct SideTable {
    spinlock_t slock;           // 自旋锁，防止多线程访问冲突。
    RefcountMap refcnts;        // 引用计数Map
    weak_table_t weak_table;    // 弱引用Map

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    bool trylock() { return slock.trylock(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<bool HaveOld, bool HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<bool HaveOld, bool HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

## spinlock_t
自旋锁，是为了解决多线程资源互斥利用提出的一种机制。是指当一个线程在获取锁的时候，如果锁已经被其它线程获取，那么该线程将循环等待，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环。
优点：不会产生线程状态的切换，线程始终active，处于用户态。相对于其他的锁，会频繁在用户态和内核态之间切换，带来极大的性能损耗。

# RefcountMap
```
// RefcountMap disguises its pointers because we 
// don't want the table to act as a root for `leaks`.
typedef objc::DenseMap<DisguisedPtr<objc_object>,size_t,true> RefcountMap;
```
这里的三个入参`DisguisedPtr<objc_object>,size_t,true`分别表示hash key、value、value==0时是否自动释放hash节点。实际上key就是OC对象，value是引用计数。

# weak_table_t
```
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;         // hash数组
    size_t    num_entries;              // 数组中存放的元素个数
    uintptr_t mask;                     // 数组长度，防止数组越界
    uintptr_t max_hash_displacement;    // hash冲突的最大次数
};
```

## weak_entry_t
```
#define WEAK_INLINE_COUNT 4
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;     // 被弱引用的对象
    union {
        // 动态数组
        struct {
            weak_referrer_t *referrers;                 // 指向referent内存地址的指针的地址的hash数组
            uintptr_t        out_of_line : 1;           // 是否使用动态hash数组标记为
            uintptr_t        num_refs : PTR_MINUS_1;    // 数组中元素个数
            uintptr_t        mask;                      // 数组长度
            uintptr_t        max_hash_displacement;     // hash冲突的最大次数
        };
        // 定长数组
        struct {
            // out_of_line=0 is LSB of one of these (don't care which)
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
```
注意WEAK_INLINE_COUNT，引用指针数量小于等于WEAK_INLINE_COUNT为定长数组，超过后就始终使用动态数组。


