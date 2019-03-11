---
title: iOS基础：UIViewController生命周期
date: 2017-01-04 11:44:02
categories: iOS基础知识
tags: 
    - iOS
    - UIViewController
---

# UIViewController生命周期
1. init
2. loadView
3. viewDidLoad
4. viewWillAppear
5. viewWillLayoutSubviews
6. viewDidLayoutSubviews
7. viewDidAppear
8. viewWillDisappear
9. viewDidDisappear
10. dealloc
<!-- more -->

# UIView layoutSubviews方法触发条件
[layoutSubviews触发条件](http://stackoverflow.com/questions/728372/when-is-layoutsubviews-called)
- init does not cause layoutSubviews to be called (duh)
- addSubview: causes layoutSubviews to be called on the view being added, the view it’s being added to (target view), and all the subviews of the target
- view setFrame intelligently calls layoutSubviews on the view having its frame set only if the size parameter of the frame is different
- scrolling a UIScrollView causes layoutSubviews to be called on the scrollView, and its superview
- rotating a device only calls layoutSubview on the parent view (the responding viewControllers primary view)
- Resizing a view will call layoutSubviews on its superview

# 在UIViewController中使用UIView时，layoutSubviews的触发时机
## 测试代码
`ViewController.m`
```
#import "ViewController.h"
#import "LKView.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    LKView *view = [[LKView alloc] initWithFrame:CGRectMake(50, 50, 100, 100)];
    [self.view addSubview:view];
    
    NSLog(@"1...");
}

- (void)viewWillLayoutSubviews
{
    [super viewWillLayoutSubviews];
    NSLog(@"3...");
}

- (void)viewDidLayoutSubviews
{
    [super viewDidLayoutSubviews];
    NSLog(@"4...");
}

- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    NSLog(@"2...");
}

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    NSLog(@"5...");
}

@end
```

`LKView.m`
```
#import "LKView.h"

@implementation LKView

- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self)
    {
        UIButton *btn = [[UIButton alloc] initWithFrame:CGRectMake(0, 0, 20, 10)];
        [self addSubview:btn];
    }
    return self;
}

- (void)layoutSubviews
{
    NSLog(@"lay out...");
}

@end
```

## 测试结果
```
2017-01-04 11:34:07.557 LKTestApp[39746:2070216] 1...
2017-01-04 11:34:07.557 LKTestApp[39746:2070216] 2...
2017-01-04 11:34:07.561 LKTestApp[39746:2070216] 3...
2017-01-04 11:34:07.561 LKTestApp[39746:2070216] 4...
2017-01-04 11:34:07.562 LKTestApp[39746:2070216] lay out...
2017-01-04 11:34:07.564 LKTestApp[39746:2070216] 5...
```

在UIViewController viewDidLayoutSubviews生命周期结束后，才会触发子view的layoutSubviews方法。