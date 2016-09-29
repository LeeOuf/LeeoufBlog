---
title: UISwitch在iOS 10中的BUG
date: 2016-09-29 13:31:22
categories: iOS
tags: 
    - iOS
    - iOS 10
    - BUG
---

# UISwitch
UISwitch是UIControl的子类，定义如下：
```
#import <Foundation/Foundation.h>
#import <CoreGraphics/CoreGraphics.h>
#import <UIKit/UIControl.h>
#import <UIKit/UIKitDefines.h>

NS_ASSUME_NONNULL_BEGIN

NS_CLASS_AVAILABLE_IOS(2_0) __TVOS_PROHIBITED @interface UISwitch : UIControl <NSCoding>

@property(nullable, nonatomic, strong) UIColor *onTintColor NS_AVAILABLE_IOS(5_0) UI_APPEARANCE_SELECTOR;
@property(null_resettable, nonatomic, strong) UIColor *tintColor NS_AVAILABLE_IOS(6_0);
@property(nullable, nonatomic, strong) UIColor *thumbTintColor NS_AVAILABLE_IOS(6_0) UI_APPEARANCE_SELECTOR;

@property(nullable, nonatomic, strong) UIImage *onImage NS_AVAILABLE_IOS(6_0) UI_APPEARANCE_SELECTOR;
@property(nullable, nonatomic, strong) UIImage *offImage NS_AVAILABLE_IOS(6_0) UI_APPEARANCE_SELECTOR;

@property(nonatomic,getter=isOn) BOOL on;

- (instancetype)initWithFrame:(CGRect)frame NS_DESIGNATED_INITIALIZER;      // This class enforces a size appropriate for the control, and so the frame size is ignored.
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder NS_DESIGNATED_INITIALIZER;

- (void)setOn:(BOOL)on animated:(BOOL)animated; // does not send action

@end

NS_ASSUME_NONNULL_END
```

方法定义中有一个`- (void)setOn:(BOOL)on animated:(BOOL)animated; // does not send action`，如注释所说不发送action。

# iOS 9下表现
接下来在iOS 9中进行验证：
```
#import "ViewController.h"

@interface ViewController ()

@property (nonatomic, strong) UISwitch *lkSwitch;
@property (nonatomic, strong) UISwitch *lkSwitch1;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _lkSwitch = [[UISwitch alloc] initWithFrame:CGRectMake(50, 100, 50, 31)];
    [_lkSwitch addTarget:self action:@selector(onSwitch:) forControlEvents:UIControlEventValueChanged];
    [self.view addSubview:_lkSwitch];
    
    UIButton *button = [[UIButton alloc] initWithFrame:CGRectMake(50, 150, 50, 31)];
    [button addTarget:self action:@selector(onButton:) forControlEvents:UIControlEventTouchUpInside];
    button.backgroundColor = [UIColor redColor];
    [self.view addSubview:button];
    
    _lkSwitch1 = [[UISwitch alloc] initWithFrame:CGRectMake(150, 100, 50, 31)];
    [_lkSwitch1 addTarget:self action:@selector(onSwitch1:) forControlEvents:UIControlEventValueChanged];
    [self.view addSubview:_lkSwitch1];
    
    UIButton *button1 = [[UIButton alloc] initWithFrame:CGRectMake(150, 150, 50, 31)];
    [button1 addTarget:self action:@selector(onButton1:) forControlEvents:UIControlEventTouchUpInside];
    button1.backgroundColor = [UIColor redColor];
    [self.view addSubview:button1];
}

- (void)onSwitch:(UISwitch *)sender {
    [sender setOn:!sender.isOn];
}

- (void)onButton:(UIButton *)sender {
    [_lkSwitch setOn:!_lkSwitch.isOn];
}

- (void)onSwitch1:(UISwitch *)sender {
    [sender setOn:!sender.isOn animated:YES];
}

- (void)onButton1:(UIButton *)sender {
    [_lkSwitch1 setOn:!_lkSwitch1.isOn animated:YES];
}

@end
```
验证结果：无论是`setOn`方法还是`setOn:_animated:_`方法，都不会触发`onSwitch`。

# iOS 10下表现
然而，在iOS 10中，测试结果显示，在`onSwitch`方法中调用`setOn`或`setOn:_animated:_`，均会再次触发`onSwitch`。

# 结论
也就是说，UISwitch中on属性的set方法在iOS 10中的表现与iOS 10以下不同，如果监听事件为`UIControlEventValueChanged`，将额外触发一次action（为什么不是不停地来回触发直到crash呢？思考...）。

# 解决方案
在action中需要改变按钮状态时，将setOn方法在主队列中异步执行。

曾提出另一个解决方案，自定义一个XXSwitch继承UISwitch，重写`setOn`和`setOn:_animated:_`方法，但是这样带来的风险是，dispatch queue以后的语句会先于`setOn`执行，若此时获取`isOn`，将无法得到预期的结果。

因此最终决定在业务逻辑层局部进行修改。