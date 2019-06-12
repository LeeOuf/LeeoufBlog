---
title: iOS基础-属性变量、实例变量、成员变量的内存泄漏
date: 2018-07-18 11:33:56
categories: iOS基础知识
tags:
    - 内存泄漏
---

直接上代码
```
@interface TestViewController () {
@private
    NSString *name1;
}

@property (nonatomic, copy) NSString *name;
@property (nonatomic, strong) TestObject *testObj;

@property (nonatomic, strong) TestObject *testObj1;

@property (nonatomic, copy) NSString *name2;
@property (nonatomic, strong) TestObject *testObj2;
@end
```
```
// 无泄漏
- (void)testBlock {
    __weak typeof(self) weakSelf = self;
    [self.testObj requestData:^(NSString *str) {
        weakSelf.name = str;
    }];
}

// 泄漏
- (void)testBlock1 {
    [self.testObj1 requestData:^(NSString *str) {
        name1 = str;
    }];
}

// 泄漏
- (void)testBlock1_2 {
    [self.testObj1 requestData:^(NSString *str) {
        self->name1 = str;
    }];
}

// 无泄漏
- (void)testBlock1_3 {
    __weak typeof(self) weakSelf = self;
    [self.testObj1 requestData:^(NSString *str) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        strongSelf->name1 = str;
    }];
}

// 泄漏
- (void)testBlock2 {
    [self.testObj2 requestData:^(NSString *str) {
        _name2 = str;
    }];
}
```

属性变量与实例变量的内存泄漏非常常见，需要关注的是成员变量name1，通过name1和self->name1均会造成内存泄漏，应与属性变量一样使用weak strong dance。
