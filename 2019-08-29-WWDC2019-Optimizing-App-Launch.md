---
title: WWDC2019 Optimizing App Launch
date: 2019-08-29 11:21:20
categories: Tech
tags:
  - iOS
  - WWDC2019
---

## What's Launch

### Launch Types
- Cold Launch: 重启后，App没有被打开过；App不存在于内存中，进程不存在。
- Warm Launch: App曾经启动过；App部分存在于内存中，进程不存在。
- Resume Launch: App被挂起，App存在于内存中，进程存在。 

速度: Resume Launch > Warm Launch > Cold Launch

### Phases of App launch
一个App的启动可以被分为以下几个部分(按时间排序):
- System Interface
- Runtime Init
- UIKit Init
- Application Init
- Initial Frame Render
- Extended

#### System Interface
System Interface又分为两部分：Dynamic Linker Loads(DYLD3)和libSystem Init

##### Dynamic Linker Loads
Dynamic Linker Loads已经发布的版本是DYLD3。DYLD负责的是加载library和framework。在iOS 13中，DYLD会缓存App的runtime dependencies，在Warm Launch的时候使用，从而提升Warm Launch的速度。
在这个阶段，App给了我们两个建议来优化加载时间:
- 移除不使用的framework，这能减少加载framework的时间
- 避免使用`DLOpen`, `NSBundleLoad`，这些方式加载的framework不能被系统给cache到。
- Hard Link所有使用的framework。

##### libSystem Init
这部分主要是初始化系统底层的一些components接口，这一阶段开发者并不能做一些什么。

#### Runtime Init
Runtime Init阶段会初始化Objc/Swift的runtime。在Objc代码中，我们通常会通过重写`[Class load]`方法来实现一些功能，这部分代码会在Runtime Init的时候被调用到。Apple的建议尽可能的把这些代码移到`[Class initialize]`中。因为`load`方法会在每次Launch时调用，`initialize`方法会在第一次使用的时候调用。

#### UIKit Init
这部分就与我们的日常开发有了比较大的关系了。在这个阶段会初始App的`UIApplication`和`UIApplicationDelegate`，如果你的App中继承了这两个类，确保他们在初始化的时候没有做耗时的工作。

#### Application Init
这个阶段涉及到的是App的Life Cycle。会调用到`UIApplicationDelegate`和`UISeceneDelegate`中的几个Life Cycle的回调方法。
- application:willFinishLaunchingWithOptions: 
- application:didFinishLaunchingWithOptions:
- applicationDidBecomeActive:
- scene:willConnectToSession:options: 
- sceneWillEnterForeground: 
- sceneDidBecomeActive:

一个原则就是不在这些方法里做耗时的操作，如果必须，想办法放到背景线程里做。

#### Initial Frame Render
这个阶段比较好理解，就是App构建用户看见的第一个视图的过程。确保第一个视图不会太复杂来减少渲染的时间。
影响渲染时间的因素主要有:
- View层级
- Autolayout

##### View
尽量使View的层级扁平化，太多的View嵌套会增加渲染的时间。

##### Autolayout
约束的数量以及是否合理会影响到Layout的时间，试着减少约束的数量并检查约束是否合理。

#### Extended
这个阶段不是每个App都有，第一次打开App时有时候我们需要加载/下载一些额外的数据，我们需要确保这部分数据都是异步加载的，然后不会影响到用户流畅的使用App。
我们可以使用`os_signpost`来measure这部分的性能。


## Measure Lauch
这一节介绍了如何度量launch time。那么首先就需要一个稳定的环境，排除其他因素的干扰。Apple给我们提供了一些方法来创造一个相对稳定的环境。

- 重启设备并放置2-3分钟 (让系统处于一个相对干净的环境下)
- 开启飞行模式或者Mock网络 (排除网络的影响)
- 使用不变的或者不使用iCloud账户 (排除iCloud的影响)
- 使用relese版本
- 度量warm launches

我们可以通过XCTest来度量我们的launch time

## Profile
Session里提了优化的Tips，其实也都是老生常谈了。启动时做最少的事情，耗时的操作放到背景线程做，减少内存的使用等等方法。

另外介绍了Instrument的新的一个模板：App Launch Template
利用这个模板，我们能在Profile的时候清楚的看到Launch的各个阶段所花费的时间，还能
