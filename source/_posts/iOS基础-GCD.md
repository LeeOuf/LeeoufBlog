---
title: iOS基础-GCD
date: 2016-08-11 20:34:23
categories: iOS基础知识
tags: 
    - 多线程
    - GCD
---

GCD是老生常谈了，不过作为一个新手这个基础乃是重中之重，光看他人的blog总是难以消化，因此自己做些笔记总结实验一下，如有偏差还望指正。

# 前提
1. 代码是顺序执行的。
2. queue是FIFO（先进先出）的数据结构。
3. 线程之间是并行的。
4. `串行队列`：任务`不能同时进行`，这意味着第n+1个任务必须等待0...n个任务全部完成后才会开始。
5. `并发队列`：任务`可以同时进行`，这意味着只要0...n个任务开始执行，那么第n+1个任务也可以开始。
6. `void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);`：将任务（也就是block）放到队列中，同步执行，也就是说在`当前线程`中执行任务，`阻塞当前线程`。
7. `void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);`：将任务放到队列中，异步执行，也就是说在`其他线程`中执行任务，`不阻塞当前线程`。（在下文的推演中，发现其并不总是在`其他线程`中执行）
（p.s. 在实际推演和API中，发现6、7两条中对于线程的描述均是错误的，block在哪个线程中运行取决于queue）

# 验证
## 1. 代码是顺序执行的。
```
    NSLog(@"step0");
    NSLog(@"step1");
```

```
    2016-08-12 22:29:49.002 MyPractice[25215:3752290] step0
    2016-08-12 22:29:49.002 MyPractice[25215:3752290] step1
```
可以看到确实是顺序执行的（严肃脸）。

## 2. queue是FIFO（先进先出）的数据结构。

## 3. 线程之间是并行的。

## 4. `串行队列`：任务`不能同时进行`，这意味着第n+1个任务必须等待0...n个任务全部完成后才会开始。
首先来看什么是`DISPATCH_QUEUE_SERIAL`。
```
    /*!
     * @const DISPATCH_QUEUE_SERIAL
     * @discussion A dispatch queue that invokes blocks serially in FIFO order.
     */
    #define DISPATCH_QUEUE_SERIAL NULL
```
该队列以先进先出的顺序串行调用blocks。

```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_SERIAL);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    for (int i = 0; i < 3; i++) {
        dispatch_async(queue, ^{
            [NSThread sleepForTimeInterval:1];
            NSLog(@"step1.%d finished! - %@", i, [NSThread currentThread]);
        });
    }
    
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 22:49:03.733 MyPractice[32046:5376787] step0 finished! - <NSThread: 0x7fac03602dd0>{number = 1, name = main}
2016-08-13 22:49:03.734 MyPractice[32046:5376787] step2 finished! - <NSThread: 0x7fac03602dd0>{number = 1, name = main}
2016-08-13 22:49:04.737 MyPractice[32046:5376906] step1.0 finished! - <NSThread: 0x7fac034a0690>{number = 2, name = (null)}
2016-08-13 22:49:05.741 MyPractice[32046:5376906] step1.1 finished! - <NSThread: 0x7fac034a0690>{number = 2, name = (null)}
2016-08-13 22:49:06.747 MyPractice[32046:5376906] step1.2 finished! - <NSThread: 0x7fac034a0690>{number = 2, name = (null)}
```
可以看到1大类中的三个任务顺序执行。

## 5. `并发队列`：任务`可以同时进行`，这意味着只要0...n个任务开始执行，那么第n+1个任务也可以开始。
```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_CONCURRENT);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    for (int i = 0; i < 3; i++) {
        dispatch_async(queue, ^{
            [NSThread sleepForTimeInterval:1];
            NSLog(@"step1.%d finished! - %@", i, [NSThread currentThread]);
        });
    }
    
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 04:21:14.000 MyPractice[27321:4331458] step0 finished! - <NSThread: 0x7f9c08c059f0>{number = 1, name = main}
2016-08-13 04:21:14.001 MyPractice[27321:4331458] step2 finished! - <NSThread: 0x7f9c08c059f0>{number = 1, name = main}
2016-08-13 04:21:15.005 MyPractice[27321:4331587] step1.1 finished! - <NSThread: 0x7f9c08f04540>{number = 3, name = (null)}
2016-08-13 04:21:15.005 MyPractice[27321:4331577] step1.0 finished! - <NSThread: 0x7f9c08d0acc0>{number = 2, name = (null)}
2016-08-13 04:21:15.005 MyPractice[27321:4331593] step1.2 finished! - <NSThread: 0x7f9c08e0b560>{number = 4, name = (null)}
```
可以看到1大类中的三个任务同时执行。

## 6. `void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);`：将任务（也就是block）放到队列中，同步执行，也就是说在`当前线程`中执行任务，`阻塞当前线程`。
### dispatch_sync任务到其它串行队列中。

```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_SERIAL);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    dispatch_sync(queue, ^{    // block1
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1 finished! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 04:28:39.621 MyPractice[27486:4339739] step0 finished! - <NSThread: 0x7f8050606ad0>{number = 1, name = main}
2016-08-13 04:28:40.622 MyPractice[27486:4339739] step1 finished! - <NSThread: 0x7f8050606ad0>{number = 1, name = main}
2016-08-13 04:28:42.623 MyPractice[27486:4339739] step2 finished! - <NSThread: 0x7f8050606ad0>{number = 1, name = main}
```
block1处于用于用户自定义串行队列中，由当前线程（主线程）执行，阻塞主线程，step2必须等待block1返回后方可开始执行。

### dispatch_sync任务到当前串行队列中。

```
    dispatch_queue_t queue = dispatch_get_main_queue();
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    dispatch_sync(queue, ^{     // block1
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1 finished! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 04:35:51.992 MyPractice[27577:4345752] step0 finished! - <NSThread: 0x7f80d8701fc0>{number = 1, name = main}
```
整体为主队列中的block0，执行完step0后等待block1返回，然后才可执行step2；同时block1插入主队列，阻塞主线程，此时主队列中还有未完成的任务block0，因此block1开始等待block1完成。
至此，block0与block1循环等待，死锁。

### dispatch_sync任务到其它并发队列中。

```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_CONCURRENT);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    dispatch_sync(queue, ^{     // block1
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1 finished! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 04:41:45.625 MyPractice[27630:4348773] step0 finished! - <NSThread: 0x7faec9403070>{number = 1, name = main}
2016-08-13 04:41:46.625 MyPractice[27630:4348773] step1 finished! - <NSThread: 0x7faec9403070>{number = 1, name = main}
2016-08-13 04:41:48.627 MyPractice[27630:4348773] step2 finished! - <NSThread: 0x7faec9403070>{number = 1, name = main}
```
step0执行完毕，block1插入用户自定义并发队列，仍在当前线程（主线程）执行，因此阻塞主线程，此时的并发队列前没有其它任务，block1正常执行完毕返回，开始执行step2。

**总结**
```
/*!
 * @function dispatch_sync
 *
 * @abstract
 * Submits a block for synchronous execution on a dispatch queue.
 *
 * @discussion
 * Submits a block to a dispatch queue like dispatch_async(), however
 * dispatch_sync() will not return until the block has finished.
 *
 * Calls to dispatch_sync() targeting the current queue will result
 * in dead-lock. Use of dispatch_sync() is also subject to the same
 * multi-party dead-lock problems that may result from the use of a mutex.
 * Use of dispatch_async() is preferred.
 *
 * Unlike dispatch_async(), no retain is performed on the target queue. Because
 * calls to this function are synchronous, the dispatch_sync() "borrows" the
 * reference of the caller.
 *
 * As an optimization, dispatch_sync() invokes the block on the current
 * thread when possible.
 *
 * @param queue
 * The target dispatch queue to which the block is submitted.
 * The result of passing NULL in this parameter is undefined.
 *
 * @param block
 * The block to be invoked on the target dispatch queue.
 * The result of passing NULL in this parameter is undefined.
 */
#ifdef __BLOCKS__
__OSX_AVAILABLE_STARTING(__MAC_10_6,__IPHONE_4_0)
DISPATCH_EXPORT DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
#endif
```
- dispatch_sync()在block结束前不会返回。
- dispatch_sync()任务到当前队列中会引起死锁。
- 系统不会retain目标队列，因为dispatch_sync()是同步执行的，它“借”走了其调度者的引用。（这里我理解为，同步执行的任务不完成不会返回，因此引用从外部跟着它进入了block内部，在任务完成后再回到外部）。
- dispatch_sync()可能会无视queue的类型，优先在当前线程执行任务。（不知在什么情况下不会在当前线程执行，此处留个心眼）。
- 参数queue － block任务提交的目标队列；如果传递进来的queue为NULL，返回结果为undefined。
- 参数block － 将要在目标队列中执行的任务；如果传递进来的block为NULL，返回结果为undedined。

## 7. `void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);`：将任务放到队列中，异步执行，也就是说在`其他线程`中执行任务，`不阻塞当前线程`。
### dispatch_async任务到其它串行队列中。

```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_SERIAL);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    dispatch_async(queue, ^{     // block1
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1 finished! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 05:06:15.123 MyPractice[27791:4366095] step0 finished! - <NSThread: 0x7fe7196022e0>{number = 1, name = main}
2016-08-13 05:06:16.129 MyPractice[27791:4366129] step1 finished! - <NSThread: 0x7fe7196a2f50>{number = 2, name = (null)}
2016-08-13 05:06:17.125 MyPractice[27791:4366095] step2 finished! - <NSThread: 0x7fe7196022e0>{number = 1, name = main}
```
执行顺序为step0 -> step1 -> step2，step1在其它线程上执行，注意step2与step0时差2秒，step1与step0时差1秒，也就是说step1与step2并行。
程序运行到step0，step1插入用户自定义串行队列，在其它线程执行，不阻塞当前线程（主线程），并立即返回，于是step2开始执行。

### dispatch_async任务到当前串行队列中。

```
    dispatch_queue_t queue = dispatch_get_main_queue();
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    dispatch_async(queue, ^{     // block1
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1 finished! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 05:10:43.516 MyPractice[27830:4369848] step0 finished! - <NSThread: 0x7fc280d07d30>{number = 1, name = main}
2016-08-13 05:10:45.518 MyPractice[27830:4369848] step2 finished! - <NSThread: 0x7fc280d07d30>{number = 1, name = main}
2016-08-13 05:10:46.524 MyPractice[27830:4369848] step1 finished! - <NSThread: 0x7fc280d07d30>{number = 1, name = main}
```
执行顺序为step0 -> step2 -> step1，step1在主线程上执行（这与我们猜想的新开线程不一致，说明我们的猜想错了，至于为什么，先放一放，后面再说），从执行时间来看任务是串行执行的。
执行完step0，step1插入主队列，不阻塞当前线程（主线程）并立即返回，开始执行step2，但是block1却必须等待block0执行完毕，因此step1在step2结束后开始执行。

### dispatch_async任务到其他并发队列中。
已经在5. xxx中实验过，3个任务在3个不同的线程上异步执行。

**总结**
```
/*!
 * @function dispatch_async
 *
 * @abstract
 * Submits a block for asynchronous execution on a dispatch queue.
 *
 * @discussion
 * The dispatch_async() function is the fundamental mechanism for submitting
 * blocks to a dispatch queue.
 *
 * Calls to dispatch_async() always return immediately after the block has
 * been submitted, and never wait for the block to be invoked.
 *
 * The target queue determines whether the block will be invoked serially or
 * concurrently with respect to other blocks submitted to that same queue.
 * Serial queues are processed concurrently with respect to each other.
 *
 * @param queue
 * The target dispatch queue to which the block is submitted.
 * The system will hold a reference on the target queue until the block
 * has finished.
 * The result of passing NULL in this parameter is undefined.
 *
 * @param block
 * The block to submit to the target dispatch queue. This function performs
 * Block_copy() and Block_release() on behalf of callers.
 * The result of passing NULL in this parameter is undefined.
 */
#ifdef __BLOCKS__
__OSX_AVAILABLE_STARTING(__MAC_10_6,__IPHONE_4_0)
DISPATCH_EXPORT DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
#endif
```
- dispatch_async()的机制是把blocks任务提交到队列中。
- dispatch_async()会立即返回而不会等待block完成。
- 目标队列的类型决定任务是按照队列里任务的顺序串行或并发地执行；串行队列之间是并发的。
**这里我的理解是，如果dispatch_async任务到一个并发队列中，那么任务就会被委派到一个新线程中执行，而如果放入一个串行队列，那么它有可能在当前线程执行，也有可能在其他线程执行。在上文实验中出现特例的是主队列，同时它也是当前队列，而主队列又是特殊的串行队列，它只在主线程上执行。这里的主队列比较特殊，不能根据它做出准确结论，我们需要再做一些实验**
```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_SERIAL);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    dispatch_async(queue, ^{     // block1.1
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1.1 finished! - %@", [NSThread currentThread]);
        
        dispatch_async(queue, ^{     // block1.2
            [NSThread sleepForTimeInterval:1];
            NSLog(@"step1.2 finished! - %@", [NSThread currentThread]);
        });
    });
    
    dispatch_async(queue, ^{     // block1.3
        [NSThread sleepForTimeInterval:1];
        NSLog(@"step1.3 finished! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:3];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 11:03:07.821 MyPractice[28927:4454751] step0 finished! - <NSThread: 0x7fbe786079a0>{number = 1, name = main}
2016-08-13 11:03:08.827 MyPractice[28927:4454778] step1.1 finished! - <NSThread: 0x7fbe78419320>{number = 2, name = (null)}
2016-08-13 11:03:09.831 MyPractice[28927:4454778] step1.3 finished! - <NSThread: 0x7fbe78419320>{number = 2, name = (null)}
2016-08-13 11:03:10.823 MyPractice[28927:4454751] step2 finished! - <NSThread: 0x7fbe786079a0>{number = 1, name = main}
2016-08-13 11:03:10.832 MyPractice[28927:4454778] step1.2 finished! - <NSThread: 0x7fbe78419320>{number = 2, name = (null)}
```
block1.1不在当前线程（主线程）中执行，而是在新线程（0x7fbe78419320）中，从当前线程切换到了新线程；
block1.2仍然在当前线程（0x7fbe78419320）中执行；
另外，block1.3也在（0x7fbe78419320）这一线程中执行，同样没有进入一个新线程；

结合实验的结果和一些blog的说法，我认为block在哪个线程取决于queue，对于每个queue来说，它和线程池有某种对应关系：
- 主队列对应着主线程；
- 并发队列对应多条线程（非主线程）；
- 串行队列对应着一条线程（非主线程）；
- dispatch_sync&dispatch_async：调度对应线程池中的线程；

接下来继续解读官方API
- 系统会保留目标队列的引用直到block结束。（queue中有任务时不会释放）
- 参数queue － block任务提交的目标队列；如果传递进来的queue为NULL，返回结果为undefined。
- dispatch_async()会代替它的调用者执行Block_copy()和Block_realease()。（block进入queue时会copy，出queue时会realease）
- 参数block － 将要在目标队列中执行的任务；如果传递进来的block为NULL，返回结果为undedined。

最后对自己提个问题，既然UI的实现没有多线程支持，那么将UI操作放入一个自定义的队列时，它在哪个线程执行呢？
```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_SERIAL);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    __weak ViewController* weakSelf = self;
    dispatch_async(queue, ^{     // block1
        NSLog(@"before step1! - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1];
        weakSelf.view.backgroundColor = [UIColor blueColor];
        NSLog(@"after step1! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 11:21:46.485 MyPractice[29211:4475426] step0 finished! - <NSThread: 0x7fd5e1403450>{number = 1, name = main}
2016-08-13 11:21:46.486 MyPractice[29211:4475474] before step1! - <NSThread: 0x7fd5e1523bb0>{number = 2, name = (null)}
2016-08-13 11:21:47.489 MyPractice[29211:4475474] after step1! - <NSThread: 0x7fd5e1523bb0>{number = 2, name = (null)}
2016-08-13 11:21:48.486 MyPractice[29211:4475426] step2 finished! - <NSThread: 0x7fd5e1403450>{number = 1, name = main}
```

```
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_SERIAL);
    
    NSLog(@"step0 finished! - %@", [NSThread currentThread]);
    
    __weak ViewController* weakSelf = self;
    dispatch_sync(queue, ^{     // block1
        NSLog(@"before step1! - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1];
        weakSelf.view.backgroundColor = [UIColor blueColor];
        NSLog(@"after step1! - %@", [NSThread currentThread]);
    });
    
    [NSThread sleepForTimeInterval:2];
    NSLog(@"step2 finished! - %@", [NSThread currentThread]);
```

```
2016-08-13 11:22:20.948 MyPractice[29246:4476879] step0 finished! - <NSThread: 0x7f8962402ee0>{number = 1, name = main}
2016-08-13 11:22:20.949 MyPractice[29246:4476879] before step1! - <NSThread: 0x7f8962402ee0>{number = 1, name = main}
2016-08-13 11:22:21.950 MyPractice[29246:4476879] after step1! - <NSThread: 0x7f8962402ee0>{number = 1, name = main}
2016-08-13 11:22:23.952 MyPractice[29246:4476879] step2 finished! - <NSThread: 0x7f8962402ee0>{number = 1, name = main}
```

可以看到仍然和之前总结的一样，在哪个线程执行仍然取决于queue的类型。
此处迷茫++，还得继续研究啊...
