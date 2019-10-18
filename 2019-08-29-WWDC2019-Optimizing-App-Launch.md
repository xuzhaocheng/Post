---
title: WWDC2019 Optimizing App Launch
date: 2019-08-29 11:21:20
categories: Tech
tags:
  - iOS
  - WWDC2019
---

这个Session是[Session 417 - Improving Battery Life and Performance](https://developer.apple.com/videos/play/wwdc2019/417/)的一个扩展。具体介绍了如何优化App的Launch Time。其中对App启动各个阶段的介绍还是挺棒的，让我们对App Launch有更深的了解。
<!-- more -->
## What's Launch
这一节介绍了App Launch的几种类型和App Launch的几个阶段。

### Launch Types
App的Launch可以分为以下三种类型:
- Cold Launch: 重启后，App没有被打开过；App不存在于内存中，进程不存在。
- Warm Launch: App曾经启动过；App部分存在于内存中，进程不存在。
- Resume Launch: App被挂起，App存在于内存中，进程存在。 

速度: Resume Launch > Warm Launch > Cold Launch

### Phases of App launch
按照时间顺序，App的Launch经历了以下几个阶段:
- System Interface
- Runtime Init
- UIKit Init
- Application Init
- Initial Frame Render
- Extended

这些阶段中，有的阶段可以被我们的程序代码影响到，有的阶段只是纯系统的工作阶段。

#### System Interface
System Interface又分为两部分：Dynamic Linker Loads(DYLD3)和libSystem Init。
这一阶段主要工作是加载dependencies和初始化App运行环境。

##### Dynamic Linker Loads
Dynamic Linker Loads最新的版本是DYLD3，在iOS 13中被引入。DYLD负责的是加载dependencies，dependencies包括framewo和library。DYLD3在第一次加载dependencies后会缓存它们，然后在Warm Launch的时候可以使用这部分缓存，从而提升Warm Launch的速度。

在这个阶段，App给了我们两个建议来优化加载时间:
- 移除不使用的framework，这能减少加载framework的时间。
- 避免使用`DLOpen`, `NSBundleLoad`等一些动态加载的函数，这些方式加载的framework不能被DYLD cache。
- Hard Link所有framework。这能保证这些framework被DYLD缓存。

关于Hard Link和Weak Link:
在*Xcode Project -> Targets -> Build Phases -> Link Binary with Libraries*下可以添加需要的Framework，`Require`的表示的就是Hard Link，而`Optional`表示的是Weak Link。使用Weak Link的原因是有时候我们需要使用到Framework只在高版本的iOS系统中存在，而Project的Deployment Target又是低版本的系统。这时候如果使用Hard Link来Link Framework，在低版本的iOS系统上运行就会Crash。

##### libSystem Init
这阶段的工作主要是初始化系统底层的一些components接口，这一阶段开发者并不能做一些什么。

#### Static Runtime Init
Runtime Init阶段负责初始化Objc/Swift的runtime。这部分的运行时间可能会被我们App的代码影响。
Apple不建议我们在这一阶段做事情，因为这会影响到启动时间。但是在Objc代码中，我们经常会通过重写`[Class load]`方法来实现一些功能(比如Swizzle)，这部分代码会在这一阶段被调用。如果非要在`load`方法里做事情，Apple建议尽可能的把这些代码移到`[Class initialize]`中。因为`load`方法会在每次Launch时调用，`initialize`方法会在第一次使用的时候调用(类似于Lazy Load)。

#### UIKit Init
这部分就与我们的日常开发有了比较大的关系了。在这个阶段会初始化App的`UIApplication`和`UIApplicationDelegate`，如果你的App中继承了这两个类，确保他们在初始化的时候没有做耗时的工作。

#### Application Init
这个阶段涉及到的是App的Life Cycle。会调用到`UIApplicationDelegate`和`UISeceneDelegate`中的几个Life Cycle的回调方法。
- `application:willFinishLaunchingWithOptions: `
- `application:didFinishLaunchingWithOptions:`
- `applicationDidBecomeActive:`
- `scene:willConnectToSession:options: `
- `sceneWillEnterForeground: `
- `sceneDidBecomeActive:`

一个原则就是不在这些方法里做耗时的操作，如果必须，想办法放到背景线程里做。

#### Initial Frame Render
这个阶段比较好理解，就是App构建用户看见的第一个视图的过程。确保第一个视图不会太复杂来减少渲染的时间。
影响渲染时间的因素主要有:
- View层级
- Autolayout

##### View
尽量使View的层级扁平化，太多的View嵌套会增加渲染的时间。

##### Autolayout
精简Constraint的数量并且检查Constraint的质量，复杂的Constraint会增加Layout的时间。

#### Extended
这个阶段不是每个App都有，第一次打开App时有时候我们需要加载/下载一些额外的数据，我们需要确保这部分数据都是异步加载的，不会影响到用户流畅的使用App。一个原则就是不阻塞主线程。
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
Session里提了优化的Tips，其实也都是老生常谈了。逃不过这么几个原则:启动时尽量做最少的工作，耗时的操作放到背景线程做，减少内存的使用等等方法。

另外介绍了Xcode 11中引入的Instrument的新模板：`App Launch Template`。
我们可以使用这个模板来帮助我们发现启动过程中的瓶颈。借助`App Launch Template`，我们能在Profile的时候清楚的看到Launch的各个阶段使用的时间，各个线程使用的情况，还能帮助我们定位问题代码。从Demo里看Instrument提供的信息还是非常多的，值得期待。

## Track
一次优化完了并不是就完事了的。在后续开发中可能会引入新的代码影响Launch Time，也可能因为种种原因导致线上的性能并不如测试时候的表现。
针对这些问题，Apple给我们提供了解决方案:
一方面，在开发阶段，我们可以使用XCTest来及时的发现Regression issue，具体来说就是编写一个监测App启动的Test，设置一个baseline。这样一来任何影响到启动时间的改动都能及时的被Track到。
另一方面，我们可以在新的Xcode Organizer中观察从用户设备上搜集的数据，来监测线上环境中用户的启动时间，帮助我们发现问题。

## Reference
[Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423)
[Frameworks and Weak Linking](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFrameworks/Concepts/WeakLinking.html)
[Weak Linking](https://nixwang.com/2017/08/26/weak-linking/)
