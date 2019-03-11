---
title: NSNotificationCenter小结
date: 2018-01-02 14:51:20
categories: iOS基础知识
tags:
    - Objective-C
    - NSNotificationCenter
---

## 线程问题
日常开发中会遇到一些异步线程修改UI的crash，很难追查，其中有一大部分可能来源于对通知的错误使用。
NSNotificationCenter是iOS开发中很常见的观察者模式，分为post和observer，其中涉及多线程的原则其实很简单，通知在哪个线程发出，observer就会在哪个线程处理。测试代码如下：
```
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    UIButton *test = [[UIButton alloc] initWithFrame:CGRectMake(50, 50, 100, 40)];
    test.backgroundColor = [UIColor blueColor];
    [test addTarget:self action:@selector(syncPost) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:test];
    
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(lisetner) name:@"asycObserverNotification" object:nil];
    });
    
    UIButton *test1 = [[UIButton alloc] initWithFrame:CGRectMake(50, 150, 100, 40)];
    test1.backgroundColor = [UIColor blueColor];
    [test1 addTarget:self action:@selector(asyncPost) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:test1];
    
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(lisetner1) name:@"asycPostNotification" object:nil];
}

// 主线程发
- (void)syncPost
{
    [[NSNotificationCenter defaultCenter] postNotificationName:@"asycObserverNotification" object:nil];
}

// 子线程收
- (void)lisetner
{
    NSLog(@"Async Observer! - %@", [NSThread currentThread]);
}

// 子线程发
- (void)asyncPost
{
    dispatch_queue_t queue = dispatch_queue_create("cn.edu.bjtu.myQueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:@"asycPostNotification" object:nil];
    });
}

// 主线程收
- (void)lisetner1
{
    NSLog(@"Async Post! - %@", [NSThread currentThread]);
    self.view.backgroundColor = [UIColor grayColor];
    [self.view layoutIfNeeded];
}
```

执行结果如下：
```
2018-01-02 14:57:56.048000+0800 LKTestApp[5192:2351336] Async Observer! - <NSThread: 0x1c407d4c0>{number = 1, name = main}

2018-01-02 14:58:09.066149+0800 LKTestApp[5192:2353929] Async Post! - <NSThread: 0x1c027dc40>{number = 5, name = (null)}
```