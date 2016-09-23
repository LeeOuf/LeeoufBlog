---
title: Objective-C基础总结
date: 2016-09-17 20:29:17
categories: iOS
tags:
    - Objective-C
---

# 内存

## 堆栈
**栈区（stack）**
由编译器自动分配释放 ，当一个函数被调用,一个stack frame(栈帧)就会被压到stack里，包含函数的参数、局部变量、返回地址等。当函数返回后,这个栈帧就会被销毁。

**堆区(heap)**
heap的空间需要手动分配。heap与动态内存分配相关,内存可以随时在堆中分配和销毁。我们需要明确请求内存分配与内存销毁。

**全局区（static）**
全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域， 未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。程序结束后由系统释放 。

**文字常量区**
常量字符串就是放在这里的。 程序结束后由系统释放。

**程序代码区**
存放函数体的二进制代码。

![内存空间](/内存空间.png)

## 引用计数

### 4个原则
- 自己生成的对象，自己持有。
- 非自己生成的对象，自己也能持有。
- 不再需要自己持有的对象时释放。
- 非自己持有的对象无法释放。

### 对象操作
| 对象操作         | Objective-C方法            |
| --------------- |:-------------------------:|
| 生成并持有对象    | alloc/new/copy/mutableCopy|
| 持有对象         | retain                    |
| 释放对象         | release                   |
| 废弃对象         | dealloc                   |

### ARC
ARC(Automatic Reference Counting): 只要还有一个变量指向对象，对象就会保持在内存中。

## 自动释放池
对于每一个Runloop， 系统会隐式创建一个Autorelease pool，这样所有的release pool会构成一个象CallStack一样的一个栈式结构，在每一个Runloop结束时，当前栈顶的Autorelease pool会被销毁，这样这个pool里的每个Object会被release。

- 对象执行autorelease方法时会将对象添加到自动释放池中。
- 当自动释放池销毁时自动释放池中所有对象作release操作。
- 对象执行autorelease方法后自身引用计数器不会改变，而且会返回对象本身。

## 属性参数
- assign: （相当于__unsafe_unretained所有权修饰符）简单赋值，用于基础数据类型和C数据类型。
- retain: （指针拷贝）拷贝一个对象，地址相同，内容相同。
- copy: （内容拷贝）拷贝一个对象，创建一个新对象，内容相同（若copy数组，则仅复制子元素的指针）。
- unsafe_unretained:若没有strong类型指针指向时，对象被释放，不自动置为nil。
- weak: （多用于解决循环引用）若没有strong类型指针指向时，对象被释放并且自动置为nil。
- strong: （ARC中默认属性类型）相当于retain。

## 实例
### 测试1
```
#import <Foundation/Foundation.h>

@interface TestARC : NSObject
@property (nonatomic, strong) NSString *string;
@property (nonatomic, assign) NSString *stringAssign;
@property (nonatomic, retain) NSString *stringRetain;
@property (nonatomic, copy) NSString *stringCopy;
@property (nonatomic, unsafe_unretained) NSString *stringUnsafe;
@property (nonatomic, weak) NSString *stringWeak;
@property (nonatomic, strong) NSString *stringStrong;
@end
```
```
#import "TestARC.h"

int main(int argc, const char * argv[]) {
    TestARC *testObj;
    
    @autoreleasepool {
        testObj = [[TestARC alloc] init];
        testObj.string = [NSMutableString stringWithFormat:@"string1"];
        testObj.stringAssign = testObj.string;
        testObj.string = nil;
    }
    
    NSLog(@"string = %@", testObj.string);
    NSLog(@"stringAssign = %@", testObj.stringAssign);
    
    return 0;
}
```
stringAssign被释放，但是没有被置为nil，形成悬垂指针（dangling pointer），程序异常终止。

### 测试2
```
#import "TestARC.h"

int main(int argc, const char * argv[]) {
    TestARC *testObj;
    
    @autoreleasepool {
        testObj = [[TestARC alloc] init];
        testObj.string = [NSMutableString stringWithFormat:@"string1"];
        testObj.stringWeak = testObj.string;
//        testObj.stringStrong = testObj.string;
        testObj.string = nil;
    }
    
    NSLog(@"string = %@", testObj.string);
    NSLog(@"stringWeak = %@", testObj.stringWeak);
    
    return 0;
}
```
```
2016-09-17 22:26:17.603 LKTestPracticeClass[31191:5393359] string = (null)
2016-09-17 22:26:17.604 LKTestPracticeClass[31191:5393359] stringWeak = (null)
```
```
2016-09-17 22:26:52.780 LKTestPracticeClass[31216:5399359] string = (null)
2016-09-17 22:26:52.781 LKTestPracticeClass[31216:5399359] stringStrong = string1
```
强指针指向对象时，内存没有被释放。

### 测试3
```
#import "TestARC.h"

int main(int argc, const char * argv[]) {
    TestARC *testObj;
    
    @autoreleasepool {
        testObj = [[TestARC alloc] init];
        testObj.string = [NSMutableString stringWithFormat:@"string1"];
        testObj.stringRetain = testObj.string;
        testObj.stringCopy = testObj.string;
        testObj.stringStrong = testObj.string;
        testObj.string = nil;
    }
    
    return 0;
}
```
```
(lldb) p testObj.string
(NSString *) $0 = nil
(lldb) p testObj.stringRetain
(__NSCFString *) $1 = 0x00000001005036c0 @"string1"
(lldb) p testObj.stringCopy
(NSTaggedPointerString *) $2 = 0x31676e6972747375 @"string1"
(lldb) p testObj.stringStrong
(__NSCFString *) $3 = 0x00000001005036c0 @"string1"
```

# Block
带有自动变量（局部变量）的匿名函数。形如int (^count)(int) =  ^int (int count){return count+1;}; 
声明部分从左到右依次为：1. 返回值类型（不可省略，无返回值时为void）；2. block名称；3. 参数列表（不可省略）。
赋值部分从左到右依次为：1. 返回值类型（无返回值时可省略，省略时根据block中return类型决定返回值，无return则为void）；2. 参数列表（无参数时可省略）；3. 表达式。

## 类型
- _NSConcretGlobalBlock: 全局静态block不会访问任何外部静态变量：这种不捕捉外界变量的block是不需要内存管理的，这种block不存在于heap或是stack而是作为代码片段存在，类似于C函数。
- _NSConcretStackBlock: 保存在栈中的block，但函数返回时被销毁，需要涉及到外界变量的block在创建的时候是在stack上面分配空间的，一旦所在函数返回，则会被摧毁。如果我们希望保存这个block或者是返回它，则需要将其放到堆上。
- _NSConcretMallocBlock: 保存在堆中的block，当引用计数为0时会被销毁。拷贝到堆后，block的生命周期就与一般的OC对象一样了，我们通过引用计数来对其进行内存管理。

## 注意点
- block会截获外部自动变量的瞬间值。
- block只能保存自动变量的瞬间值，而不能改写。若想改写，可以为自动变量添加__block修饰符。此外，静态变量、全局变量、静态全局变量这些全局区中的变量可以在block中改写值。
- 使用block时很容易出现两个对象互相持有、循环引用的情况，这会导致内存泄漏，可以通过如`__weak typeof(self) weakSelf = self;`新建一个指针的方式解决，在block内部使用__weak类型的对象。
- block属性的声明，首先需要用copy修饰符，因为只有copy后的block才会在堆中，栈中的block的生命周期是和栈绑定的。在ARC下,即使你声明的修饰符是strong，实际上效果是与声明为copy一样的。因此在ARC情况下,创建的block仍然是NSConcreteStackBlock类型，只不过当block被引用或返回时，ARC帮助我们完成了copy和内存管理的工作。

## 实例
```
#import <Foundation/Foundation.h>

@interface TestBlock : NSObject
typedef NSString *(^testBlockStruct)(NSString *);
@property (nonatomic, strong) testBlockStruct completionBlock;

- (void)handleBlock:(NSString *)string;
@end
```
```
#import "TestBlock.h"

@implementation TestBlock
- (void)handleBlock:(NSString *)string {
    if (self.completionBlock) {
        NSLog(@"%@", self.completionBlock(string));
    }
}
@end
```
```
#import "TestBlock.h"

int main(int argc, const char * argv[]) {
    TestBlock *testBlock = [[TestBlock alloc] init];
    
    NSString *tempString = @"Baidu";
    testBlock.completionBlock = ^NSString *(NSString *string){
        // 使用外部的局部变量tempString
        NSLog(@"Block is here with %@ at %@.", string, tempString);
        return @"Block is completed.";
    };
    // 改变局部变量tempString，打印出的block截获了其瞬间值"Baidu"
    tempString = @"Beijing";
    
    [testBlock handleBlock:@"LeeOuf"];
    return 0;
}
```
```
2016-09-17 22:33:05.425 LKTestPracticeClass[31278:5428415] Block is here with LeeOuf at Baidu.
2016-09-17 22:33:05.426 LKTestPracticeClass[31278:5428415] Block is completed.
```

# GCD
GCD(Grand Central Dispatch)是异步执行任务的技术之一。允许开发者将自定义任务追加到适当的Dispatch Queue。

## Dispatch Queue
- DISPATCH_QUEUE_SERIAL: 串行队列，FIFO，队列中0…n任务执行结束后，n+1个任务才可以开始。
- DISPATCH_QUEUE_CONCURRENT: 并发队列（并发不一定能并行），FIFO，队列中0…n任务开始执行后，n+1个任务才可以开始。

获取Dispatch Queue方法: 
- dispatch_get_main_queue()
- dispatch_get_global_queue()
- dispatch_queue_create()

## 方法列表
![queue.h](/GCD01.png)
![group.h](/GCD02.png)
![once.h](/GCD02.png)

# 补充
1. MRC不能使用weak，使用什么替代？
2. NSString为什么要使用copy?
3. block底层forwarding实现
4. 那些部分不能使用async操作
5. ARC和MRC下的循环引用
6. 如何检测VC的循环引用
7. Category的好处，Category中是否可以加property？
8. 类别和扩展的区别？为啥要使用扩展
9. Category中一定添加property，如何实现？
10. Category中添加和原有类的方法，是否覆盖？  
-------------------------------- 
1. assign
2. 因为NSString可以持有NSMutableString对象，如果将NSMutableString对象赋值给一个NSString，使用copy可以防止前者的修改引起后者值发生变化（NSString应该是immutable）。
3. `__Block_byref_val_0`结构体实例有一个成员变量`__forwarding`持有只想该实例自身的指针。
static void _Block_byref_assign_copy(void *dest, const void *arg, const int flags) {
    struct Block_byref **destp = (struct Block_byref **)dest;
    struct Block_byref *src = (struct Block_byref *)arg;
    // src points to stack
    struct Block_byref *copy = (struct Block_byref *)_Block_allocator(src->size, false, isWeak);
    copy->forwarding = copy; // patch heap copy to point to itself (skip write-barrier)
    src->forwarding = copy;  // patch stack to point to heap copy
    copy->size = src->size;
    // assign byref data block pointer into new Block
    _Block_assign(src->forwarding, (void **)destp);
}
根据以上代码可看到，该函数在堆上新建了一个对象copy，将堆上copy对象的forwarding和栈上原始对象的forwarding都指向copy对象。以保证无论是在堆上还是在栈上，我们通过forwarding访问到的都是同一个对象。
4. UI部分。
5. ARC中可以新建一个__weak修饰符修饰的指针（如__weak typeof(self) weakSelf=self;），防止两个对象互相持有；MRC可以用__block修饰（如__block typeof(self) weakSelf = self;）。
6. 
(1) Product->Profile，查看Leaks，如：
![Leaks](/调试01.png)
![Leaks结果](/调试02.png)
(2) lldb直接print，查看对象是否为nil。
7&9. 优点：不使用继承而为现有类添加新方法。
Category用常规方法添加property即使实现了get set方法，在调用时也会报错，但是可以通过objc_getAssociatedObject和objc_setAssociatedObject这两个函数或@dynamic在运行时添加property。
8&10. （1）类别：类别中无法添加新的实例变量；类别中的方法与现有方法的命名冲突时，类别中的方法将会取代初始方法；类别主要用于将类的实现分散到不同的文件中、创建对私有方法的前向引用、向对象添加非正式协议。
（2）扩展：一种匿名的类别；私有的属性和方法可以通过扩展的方式声明；扩展中定义的方法必须在@implementaion中实现。
