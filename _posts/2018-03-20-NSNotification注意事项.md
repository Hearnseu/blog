---
layout:     post                   
title:      NSNotification 注意事项             
subtitle:   
date:       2018-03-20           
author:     Hearnseu                      
header-img: img/post-bg-2015.jpg    
catalog: true                       
tags:                              
    - iOS
---



## 1. 在主线程注册通知

```objective-c
 [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(log) name:@"pst" object:nil];
```
`log`方法的执行会在发送通知的线程执行。如在子线程发通知，`log`的执行就在子线程执行。

主线程注册的通知，无论什么线程发送，都能收到。

换另一种注册方式：

```objective-c
//设置执行队列
 [[NSNotificationCenter defaultCenter] addObserverForName:@"pst" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * _Nonnull note) {
        [self log];
    }];
```
那么无论怎么发通知，都会在主线程处理。

## 2. 在子线程注册通知

```objective-c
//新建线程并在5s后发通知
 _thd = [[NSThread alloc] initWithTarget:self selector:@selector(mythread) object:nil];
    _thd.name = @"he";
    [_thd start];
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
         [[NSNotificationCenter defaultCenter] postNotificationName:kNotificationName object:nil];
    });
   
- (void)mythread{
    
     //在子线程中注册通知
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(log) name:kNotificationName object:nil];
    _timer = [NSTimer scheduledTimerWithTimeInterval:3 target:self selector:@selector(say) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:_timer forMode:NSRunLoopCommonModes];
    [[NSRunLoop currentRunLoop] run];
}
    
```

注意：通知注册需放在`[[NSRunLoop currentRunLoop] run];
`前，否则无效。

同样，子线程注册的通知，无论什么线程发送通知，该子线程都能收到。

主线程发送的通知，`log`方法依然在主线程执行，其他线程发送的通知，在该发送线程中执行。



