---
layout: post
title: 'Animation去哪儿了'
categories: Tech
tags:
  - iOS
---

前几天遇到一个十分诡异的Bug，有用户在使用App的过程中，遇到了所有动画突然都消失的问题。
没有Log文件，没有固定的重现步骤，一个Random的issue。
刚接到的时候内心是懵逼的，毫无头绪，拿了几个case试了几遍并没有重现问题。又换了几个case，终于有一个case能不稳定的重现了，几率还算比较高。
可是还是不知道从何下手。思考了一下，发现是所有的动画都消失了，肯定不是某个`UIView`或者某个动画的问题。
<!-- more -->
尝试Google了一下，找到了个关键的线索，`[UIView setAnimateionEnable:]`。`UIView`的这个类方法可以控制所有动画的开启或者关闭。
八九不离十，一定是是这个变量被设置错了。全局搜索了工程里调用这个方法的地方，全都打上断点，重新试了case。断点没跑到，还是出现了动画消失的问题。那么一定是哪个系统调用，调用了这个方法，导致变量被设置错了。这就比较难办了，`UIKit`并不开源，我们没办法找出所有调用`setAnimateionEnable:`的地方。

没办法，只能祭出Hook大法了，自己动手写个`UIView`的`Category`来Hook这个方法。

```objc
#import "UIView+Hook.h"
#import <objc/runtime.h>
@implementation UIView (Hook)

void SwizzleClassMethod(Class c, SEL orig, SEL new) {

    Method origMethod = class_getClassMethod(c, orig);
    Method newMethod = class_getClassMethod(c, new);

    c = object_getClass((id)c);

    if(class_addMethod(c, orig, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
        class_replaceMethod(c, new, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
    else
        method_exchangeImplementations(origMethod, newMethod);
}


+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        SwizzleClassMethod([self class], @selector(setAnimationsEnabled:), @selector(sw_setAnimationsEnabled:));
    });
}

+ (void)sw_setAnimationsEnabled:(BOOL)enabled {
    [[self class] sw_setAnimationsEnabled:enabled];
}

@end
```

打个断点在`sw_setAnimationsEnabled:`里看一看堆栈究竟是谁最后将动画禁止了，又跑了一遍case。
Emmmmm...茫茫多的调用，断点跑了无数遍，一头包。想一想只要保留住最后一次设置为`NO`的调用堆栈即可。让我们再来加些代码。

```objc
#import "UIView+Hook.h"
#import <objc/runtime.h>

@implementation UIView (Hook)

static NSArray * Callstacks;
static BOOL LastStatus = YES;

void SwizzleClassMethod(Class c, SEL orig, SEL new) {

    Method origMethod = class_getClassMethod(c, orig);
    Method newMethod = class_getClassMethod(c, new);

    c = object_getClass((id)c);

    if(class_addMethod(c, orig, method_getImplementation(newMethod), method_getTypeEncoding(newMethod)))
        class_replaceMethod(c, new, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
    else
        method_exchangeImplementations(origMethod, newMethod);
}


+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        SwizzleClassMethod([self class], @selector(setAnimationsEnabled:), @selector(sw_setAnimationsEnabled:));
    });
}

+ (void)sw_setAnimationsEnabled:(BOOL)enabled {
    if (!enabled) {
        if (LastStatus) {
            Callstacks = [NSThread callStackSymbols];
        }
    } else {
        Callstacks = nil;
    }

    LastStatus = enabled;
    [[self class] sw_setAnimationsEnabled:enabled];
}

+ (NSString *)printCallstack {
    NSMutableString *ret = [[NSMutableString alloc] init];
    for (id line in Callstacks) {
        [ret appendString:line];
        [ret appendString:@"\n"];
    }
    return ret;
}

@end

```

用静态变量`LastStatus`来记录上一次设置的值，用静态变量`Callstacks`来记录最后一次设置为`NO`的调用堆栈，让我们来追踪一下是谁设置了`NO`而没有还原。

又跑了一遍case，成功触发了动画消失，抓紧看一下`Callstacks`里的堆栈信息，迅速定位到了出问题的代码：工程中有一处调用了`[UIView performWithoutAnimation:]`方法，希望在做某些操作的时候禁用`CoreAnimation`的隐式动画。而这个方法会先记录`AnimationsEnabled`的值，并将`AnimationsEnabled`设置为`NO`禁用动画，然后执行block，最后将`AnimationsEnabled`恢复到之前的状态。
很显然，是最后恢复的时候恢复到了错误的值，那么为什么会恢复错了呢？瞄了一眼堆栈，发现这个操作是在背景线程做的。
一切都清楚了：
1. 在多线程的环境下调用了`[UIView performWithoutAnimation:]`方法。这个方法属于`UIKit`，它并不是线程安全的。在线程A中调用了`[UIView performWithoutAnimation:]`，记录`AnimationsEnabled`为`YES`，然后全局变量`AnimationsEnabled`被设置为了`NO`，接着执行block。
2. 但是在线程A的block执行完之前，线程B调用了`[UIView performWithoutAnimation:]`记录了此刻的`AnimationsEnabled`为`NO`，然后执行block。
3. 线程A的block先执行完毕，恢复`AnimationsEnabled`为`YES`，线程B的block后执行完毕，恢复`AnimationsEnabled`为
`NO`。此时`AnimationsEnabled`的状态就错了，造成了所有的动画消失。

这个issue的根本原因还是在背景线程调用`UIKit`的API。我们应该在编码的时候就严格控制这种情况的发生，否则会发生一些看起来十分奇怪的问题。`Xcode`已经为我们提供了实用的工具来帮我们检查背景线程调用`UIKit`API的情况。
勾上`Scheme`->`Diagnostics`->`Runtime API Checking`->`Main Thread Checker`就能替我们检查这类问题，但是由于工程比较庞大，很多人并不会非常在意这类提示，或者是看见了提示，但是感觉能work就选择忽略它。我个人还是建议把`Pause on issues`也打开，这样能强制我们去解决这类有潜在风险的调用。