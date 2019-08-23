---
layout: post
title: 'WWDC2019 - Improving Battery Life and Performance'
categories: Tech
tags:
  - iOS
  - WWDC2019
---

这一个Session主要是讲测试工具。介绍了在App的各个阶段，我们可以使用Apple提供的各种工具来帮助我们做Testing，发现一些潜在的问题。
首先我们要知道，需要Improve Battery Life and Performance就不可避免的要搜集一些Metrics(指标)，只有有了Metrics，才能对比出好坏。
<!-- more -->
开头介绍了一个App从开发到上线会经历三个阶段：
`Development and Testing` -> `Beta` -> `Public Release` 
在这三个阶段，我们可以利用Apple提供的不同的工具来搜集metrics，从而达到发现问题解决问题的目的。
Session里介绍了三种不同的搜集Metrics的工具
- XCTest Metrics (开发阶段)
- MetricsKit (测试和线上阶段)
- Xcode Metrics Organizer (线上阶段)

今年，Apple添加了两大类的Metrics，分别是Battery和Performance。
对于Battery可以针对以下一些service分别搜集Metrics:
- Processing
- Location
- Display
- Networking
- Accessories
- Multimedia
- Camera

对于Performance可以针对以下一些case分别搜集Metrics:
- Hangs
- Disk
- Application Launch
- Memory
- Custom Intervals


## Measuring App Impact during Development and Testing
在开发阶段我们可以借助的是XCTest Metrics。
在开发阶段，我们会编写一些Unit Testing和UI Testing来帮助我们发现Regression issue。对于性能问题，我们也可以使用XCTest来帮助我们发现问题，XCTest Metrics就是这样的工具。
在Xcode 11之前，XCTest就已经支持了一部分的性能测试，但是我们只能设置一些baseline来进行性能方面的监测。而在Xcode 11中，增加了一些新的Metrics:
- CPU
- Memory
- Storage
- Clock
- OSSignpost
- Custom metrics

现在，我们可以通过这样一段代码来测试App的启动时间：
```swift
func testAppLaunchTime() {
    measure(metrics: [XCTOSSignpostMetric.applicationLaunch]) {
        XCUIApplication().launch()
    }
}
```
需要注意的是，`XCTOSSignpostMetric.applicationLaunch`在老版本的Xcode里是不存在的，你需要升级到最新版本的Xcode，我使用的是*Xcode 11.0 beta 6*。

和以前一样，我们可以设置自己的baseline，可以观察每次测试的结果。
![Performance Result](performance-result.png)

我们也可以同时监测多种Metrics:
```swift
func testSomePerformance() {
    let app = XCUIApplication()
    measure(metrics: [XCTClockMetric(),
                        XCTMemoryMetric(application: app), XCTCPUMetric(application: app)]) {
        // do something
    }
}
```

我们可以选择不同的Metrics观测结果：
![Performance Result](multi-performance-result.png)

### Measuring App Impact in the Field 
有些时候，我们需要搜集用户在使用App时的性能指标，以往各大厂商的做法可能是自己实现一套计时打点系统。在关键处埋点，搜集一定数据后再传回到服务器上以供分析。
现在我们可以使用Apple提供的`MetricKit`来帮助我们实现这样的功能。

```swift
// Adopting MetricKit to receive metrics
import MetricKit
// 1. Conform to MXMetricManagerSubscriber protocol
class MySubscriber: NSObject, MXMetricManagerSubscriber {
    var metricManager: MXMetricManager?
    override init() {
        super.init()
        // 2. Initialize MetricManager
        metricManager = MXMetricManager.shared
        // 3. Subscribe for metrics
        metricManager?.add(self)
    }

    deinit {
        metricManager?.remove(self)
    }

    // Adopting MetricKit to receive metrics
    // 4. Implement delegate method
    func didReceive(_ payload: [MXMetricPayload]) {
        for metricPayload in payload {
            // 5. Consume metric payloads ......
        }
    }
}
```

我们需要在App的工程内编写这样一个类，它遵循了`MXMetricManagerSubscriber`协议，`didReceive`方法会在系统产生一个新的`MXMetricPayload`时候调用。我们可以对这些Payload进行操作，比如上传到服务器，写入到本地文件等等。需要注意的是这个方法会在背景线程调用。

Session里提到了会对每天搜集的Metrics做一次总结然后返回给设备，所以这个回调方法也会在每天回调一次。
> The system calls this method at most once per day.

使用Xcode attach到设备上后，可以通过*Debug->Simulate MetricKit Payloads*来查看搜集的指标数据。

### Metrics for Critical Code Sections
另一个这次提供的工具是`mxSignposts`，它也属于`MetricKit`，利用这个工具，我们可以方便的测量我们关键逻辑的性能参数。比如处理图像、处理音视频等一些耗时的工作，我们希望监测一些指标。
使用的方法也很简单，打点，就完了。

```swift
// Example: Collect metrics for critical code sections using mxSignposts
// Each mxSignpost (build over os_signposts) snapshots CPU time, memory and logical Writes
// Currently, custom metadata isn’t supported with mxSignposts
// 1. Create log handle using MetricKit’s makeLogHandle method
let photosLogHandle : OSLog = MXMetricManager.makeLogHandle(category: "Photos")
// 2. Drop mxSignpost around critical code sections
mxSignpost(.begin, log: photosLogHandle, name: "SavePhoto")
SavePhoto() // Application code
mxSignpost(.end, log: photosLogHandle, name: "SavePhoto")
```

但是这个API似乎只是对之前的`os_signpost`方法的一种封装？


## Xcode Metrics Organizer
在*Window->Organizer->Metrics*中可以找到这个Organizer。
这实际上是一个MetricKit搜集的指标的可视化工具，它可以通过多种维度来展示搜集到的Metrics，便于我们从中发现问题。
需要注意的是，只有到搜集够一定Metrics后才会呈现出图表，数据不够是不行的。

