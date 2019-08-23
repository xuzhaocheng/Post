---
layout: post
title: 'LKImageKit源码解读（Core）'
categories: Tech
tags: 
  - iOS
---

# 写在最前面
偶然间在`Github`上发现了腾讯的开源项目`LKImageKit`，看着描述觉得很厉害的样子，突然来了兴致，来分析它的代码和架构。
这篇文章不会在`UI`层次上有过多的纠缠，主要目的是分析其workflow和系统架构。对应的项目文件主要是`Core`目录下的文件。
这篇文章**不会讲**图片是如何加载的、图片是怎样解码的、缓存策略是怎样的、每种Processor是如何处理的。
<!-- more -->
# 系统架构
`LKImageKit`的架构和`SDWebImage`的架构有一些相像。以一个Manager（`LKImageManager`）为中心，来管理Image的加载。同时还有`LKImageDecoderManager`，`LKImageCacheManager`，`LKImageLoaderManager`，`LKImageProcessorManager`几个外围的Manager来协助`LKImageManager`工作。每个`Manager`不直接处理相关工作，而是将工作派发给具体的类去处理。比如`LKImageDecoderManager`会将Decode的工作交给`LKImageDecoder`的子类去做，而这些子类可以由我们自己实现，这也就是官方在其文档中提到的有高度的可扩展性。
`LKImageKit`与`SDWebImage`有一点很大的不同是，在`LKImageKit`中引入了`Request`的概念(`LKImageRequest`)。在各个模块中都能看见`LKImageRequest`的影子，在整个workflow中有着重要的作用。
另外`LKImageKit`实现了`LKImageView`，通过继承`UIView`来实现相关的逻辑功能，与`SDWebImage`也略有不同。
下面这个类图能帮助我们对`LKImageKit`的总体架构有个比较初步的认识：

![Simple Class Diagram](LKImageKit-Simple-Class-Diagram.png)


# 源码分析

`LKIImageRequest`在各个模块中都有涉及，先分析它对于我们理解其他模块会有帮助。

## LKIImageRequest

先来看一看`LKImageRequest.h`中都有哪些属性和方法。
头文件中的属性几乎都有对应的注释，命名也较为规范，在这就不一一列举了。

```objc
//Identify of request. Indicate requests is equal or not. Always use in memory cache
//Request的标识符。被用于内存缓存
@property (nonatomic, strong) NSString *identifier;

//Key for loading file. Indicate requests's data is from whitch source. Always use in disk cache
//标识了request的数据是从什么源来的。被用于磁盘缓存
@property (nonatomic, strong) NSString *keyForLoader;
rogressive;

@property (nonatomic, assign, readonly) LKImageRequestState state;

//Default is 0
//优先级(Request支持优先级)
@property (atomic, assign) NSOperationQueuePriority priority;

//Processor will progress the image before display
//用户可以自定义的processor队列，在图片被展示前，会对图片逐一执行队列中的每一个processor
@property (nonatomic, strong) NSArray<LKImageProcessor *> *customProcessorList;

//Indicate the request is synchronized or asynchronized.
//If true, all operation for request will be synchronized.
//If false, cache is synchronized but LKImageManager/LKImageProcessorManager/LKImageLoaderManager has its own queue.
//是否为同步请求
//如果是，所有的请求的操作都会同步执行
//如果否，cache是同步的，但是LKImageManager/LKImageProcessorManager/LKImageLoaderManager都会有自己的队列
@property (nonatomic, assign) BOOL synchronized;
```

可能你会发现工程中还存在一个头文件`LKImageRequest+Private.h`。这个头文件中声名的变量和方法，只被工程内部所使用，在我们外部使用框架的时候是不需要用到的。
其实，仔细观察后你会发现，这些变量主要的作用是存储信息，这些信息随着`LKImageRequest`在各个模块之间传递而传递。


`LKImageRequest.m`的实现也不是很复杂，我们逐一的来分析每个方法，一些简单的`getter`和`setter`在这里被略去。


```objc
- (instancetype)init
{
    if (self = [super init])
    {
        self.cacheEnabled = YES;
        atomic_fetch_add(&LKImageTotalRequestCount, 1);
    }
    return self;
}

- (void)dealloc
{
    atomic_fetch_sub(&LKImageTotalRequestCount, 1);
}
```

`init`和`dealloc`。默认启用了`cache`，`LKImageTotalRequestCount`是用来统计`request`数量的，用来分析性能。


```objc
- (id)copy
{
    LKImageRequest *request = [[[self class] alloc] init];
    Class cls               = [self class];
    while (cls != [NSObject class])
    {
        unsigned int numberOfIvars = 0;
        Ivar *ivars                = class_copyIvarList(cls, &numberOfIvars);
        for (const Ivar *p = ivars; p < ivars + numberOfIvars; p++)
        {
            Ivar const ivar = *p;
            NSString *key   = [NSString stringWithUTF8String:ivar_getName(ivar)];
            if (key == nil)
            {
                continue;
            }
            if ([key length] == 0)
            {
                continue;
            }

            id value = [self valueForKey:key];
            
            @try
            {
                [request setValue:value forKey:key];
            }
            @catch (NSException *exception)
            {
            }
        }
        if (ivars)
        {
            free(ivars);
        }

        cls = class_getSuperclass(cls);
    }
    return request;
}
```
`copy`方法用了`runtime`的机制来实现，大概是为了避免在每个子类都重写一遍`copy`。拿到`Ivars`的列表，逐一复制，接着找父类，继续复制，直到基类`NSObject`。

```objc
- (instancetype)createSuperRequest
{
    LKImageRequest *request = [self copy];
    request.requestList     = [NSMutableArray array];
    request.managerCallback = nil;
    request.loaderCallback  = nil;
    [request addChildRequest:self];
    return request;
}
```
创建一个父亲`request`，在`LKImageManager`和`LKImageLoaderManger`里会用到，具体的作用后面会说。这里只要知道`request`在使用中会生成一颗树。具体是如何组织的在后面会说到。


```objc
- (BOOL)isEqual:(id)object
{
    if ([object isKindOfClass:[LKImageRequest class]])
    {
        if ([self.identifier isEqualToString:((LKImageRequest *) object).identifier])
        {
            return YES;
        }
    }
    return NO;
}

- (void)generateIdentify
{
    if (self.processorList.count == 0)
    {
        self.identifier = self.keyForLoader;
    }
    self.identifier = [NSString stringWithFormat:@"%@:%@", self.keyForLoader, [LKImageProcessorManager keyForProcessorList:self.processorList]];
}
```
`isEqual:`使用`identifier`来判断两个`request`是否相等。相关的还有`generateIdentify`方法，如果没有`processorList`直接使用`keyForLoader`作为`identifier`，否则是一个`keyForLoader`和`processorList`的组合串。这也容易想到，因为即使`keyForLoader`相同，但`processorList`不同，最后的Image其实是不同的。


```objc
- (void)reset
{
    self.error                        = nil;
    self.progress                     = 0;
    self.isCanceled                   = NO;
    self.isStarted                    = NO;
    self.isFinished                   = NO;
    self.imageManagerCancelOperation  = nil;
    self.loaderManagerCancelOperation = nil;
    self.decodeOperation              = nil;
}

```
重置一个`request`，将其恢复到初始状态，可以理解为还没开始执行之前。


```objc
- (void)managerCallback:(UIImage *)image isFromSyncCache:(BOOL)isFromSyncCache
{
    if (self.managerCallback)
    {
        self.managerCallback(self, image, isFromSyncCache);
    }
    for (LKImageRequest *request in self.requestList)
    {
        if (request.progress >= 1 || request.error)
        {
            request.requestEndDate = [NSDate date];
        }

        if (request.managerCallback)
        {
            request.managerCallback(request, image, isFromSyncCache);
        }
    }
}

- (void)invokePreloadCallback
{
    if (self.preloadCallback)
    {
        self.preloadCallback(self);
        if (self.progress>=1 || self.error)
        {
            self.preloadCallback = nil;
        }
    }
    for (LKImageRequest *request in self.requestList)
    {
        if (request.preloadCallback)
        {
            request.preloadCallback(self);
        }
    }
}

- (void)loaderCallback:(UIImage *)image
{
    [self invokePreloadCallback];
    if (self.loaderCallback)
    {
        self.loaderCallback(self, image);
    }
    for (LKImageRequest *request in self.requestList)
    {
        if (request.loaderCallback)
        {
            request.loaderCallback(request, image);
        }
    }
}
```
`managerCallback:isFromSyncCache:`
此方法会在`request`完成或者进行中时后回调，`LKImageManager`负责回调这个方法。如果整个请求成功了，`image`参数返回图像；如果失败了，则会是`nil`。
同时，还会在`request`树上调用其孩子`request`的`managerCallback`回调。
实际上，成功的回调只有两处。一处在`LKImageManager`从`cache`中加载成功Image，此时`isFromSyncCache`为`YES`；另一处在`loader`加载成功后，此时`isFromSyncCache`为`NO`。

`invokePreloadCallback:`
此方法回调了`preloadCallback`，和`managerCallback:isFromSyncCache:`一样，也会调用孩子`request`的`preloadCallback`。

`loaderCallback:`
此方法在`loader`完成load后调用，回调`loaderCallback`，和`managerCallback:isFromSyncCache:`一样，也会调用孩子`request`的`loaderCallback`。

这三个方法的`callback`实际代码都不是由`LKImageRequest`决定的，来自于各个`Manager`，是workflow中不可缺少的组成部分。可以理解为`request`带着这些`callback`在模块间流动，实际上它只是一个载体，何时被调用以及执行什么代码并不由自身决定。


```objc
- (void)addChildRequest:(LKImageRequest *)request
{
    [_requestList addObject:request];
    request.superRequest = self;
    if (request.cacheEnabled)
    {
        self.cacheEnabled = YES;
    }
    if (request.supportProgressive)
    {
        self.supportProgressive = YES;
    }
    if (request.synchronized)
    {
        self.synchronized = YES;
    }
    if (request.priority > self.priority)
    {
        self.priority = request.priority;
    }
    if (CGSizeEqualToSize(self.preferredSize, CGSizeZero))
    {
        self.preferredSize = request.preferredSize;
    }
    if (request.loaderCallback)
    {
        self.loaderCallbackCount++;
    }
}

- (void)removeChildRequest:(LKImageRequest *)request
{
    NSUInteger index = [self.requestList indexOfObject:request];
    if (index != NSNotFound)
    {
        [self removeChildAtIndex:index];
    }
}

- (void)removeChildAtIndex:(NSUInteger)index
{
    @synchronized(self)
    {
        LKImageRequest *request = _requestList[index];
        [_requestList removeObjectAtIndex:index];
        request.superRequest = nil;
        
        if (request.loaderCallback)
        {
            self.loaderCallbackCount--;
        }
        NSInteger priority = -1000;
        for (LKImageRequest *request in self.requestList)
        {
            if (request.priority > priority)
            {
                priority = request.priority;
            }
        }
        self.priority = priority;
    }
}

```
`Child Request`相关代码，需要注意的是，父亲`request`的优先级是取孩子`request`中优先级最高的那个。



在来看看`LKImageRequest`的两个子类，`LKImageURLRequest`和`LKImageImageRequest`。

`LKImageURLRequest`用来请求本地或者是远程的图片。

它比父类多了几个构造方法，重写了`description`方法。
```objc
+ (instancetype)requestWithURL:(NSString *)URL key:(NSString *)key
{
    LKImageURLRequest *request = [[self alloc] init];
    request.URL                = URL;
    if (!key)
    {
        key = [LKImageUtil MD5:URL];
    }
    request.keyForLoader       = key;
    return request;
}

+ (instancetype)requestWithURL:(NSString *)URL key:(NSString *)key supportProgressive:(BOOL)supportProgressive
{
    LKImageURLRequest *request = [[self alloc] init];
    request.URL                = URL;
    if (!key)
    {
        key = [LKImageUtil MD5:URL];
    }
    request.keyForLoader       = key;
    request.supportProgressive = supportProgressive;
    return request;
}
```
创建时需要提供`URL`，如果没有`key`则会使用`URL`的`MD5`值作为`key`。对应的`loader`有
+ `LKImageLocalFileLoader`负责load本地图片
+ `LKImageNetworkFileLoader`负责load网络图片
+ `LKImagePhotoKitLoader`负责load相册图片


`LKImageImageRequest`用来请求内存图片。
```objc
+ (instancetype)requestWithImage:(UIImage *)image
{
    return [self requestWithImage:image key:nil];
}

+ (instancetype)requestWithImage:(UIImage *)image key:(NSString *)key
{
    LKImageImageRequest *request = [[LKImageImageRequest alloc] init];
    request.image                = image;
    request.keyForLoader         = [NSString stringWithFormat:@"%p",image];
    request.synchronized         = YES;
    return request;
}

- (instancetype)init
{
    if (self = [super init])
    {
        self.cacheEnabled = NO;
    }
    return self;
}

- (BOOL)isEqual:(id)object
{
    if ([object isKindOfClass:[LKImageImageRequest class]])
    {
        LKImageImageRequest *request = (LKImageImageRequest *) object;
        return request.image == self.image;
    }
    return NO;
}

- (NSString *)identifier
{
    return [NSString stringWithFormat:@"%p", self];
}
```
使用`image`来构造一个`LKImageImageRequest`，看起来似乎很傻，估计是为了走框架这套流程造出来的吧。对应的`loader`是`LKImageMemoryImageLoader`，也只解析这一种`LKImageRequest`。


## LKImageView

照例先看头文件`LKImageView.h`里都有些什么（只列出部分）

```objc
@interface LKImageView : UIView

@property (nonatomic, strong) UIImageView *imageView;

@property (nonatomic, weak) id<LKImageViewDelegate> delegate;

@property (nonatomic, strong) LKImageRequest * _Nullable loadingImageRequest;  //image request when loading
@property (nonatomic, strong) LKImageRequest * _Nullable failureImageRequest;  //image request when failed
@property (nonatomic, strong, readonly) UIImage * _Nullable presentationImage; //final image to be display.
@property (nonatomic, strong) LKImageRequest * _Nullable request;              //image request send to LKImageManager and get UIImage from it

@property (nonatomic, strong) LKImageManager *imageManager;

@end
```
可以发现`LKImageView`实际上是继承自`UIView`，并且添加了一个`UIImageView`作为`subview`。`UI`相关的先略过，来看看`LKImageView`有着三个`LKImageRequest`：

+ `loadingImageRequest` 正在载入显示的图片的request
+ `failureImageRequest` 载入失败显示的图片的request
+ `request` 真正需要显示的图片的request

其实就是指明了三种状态的图片的来源。

还有一个`delegate`来通知图片加载的状态，定义如下：
```objc
@protocol LKImageViewDelegate <NSObject>
@optional

- (void)LKImageViewImageLoading:(LKImageView *)imageView request:(LKImageRequest *)request;
- (void)LKImageViewImageDidLoad:(LKImageView *)imageView request:(LKImageRequest *)request;

@end
```

`imageManager`指明这些`request`会交由谁去加载，你也可以自己制定`Manager`（但是一般不需要）。

再来看看`LKImageView.m`
略过`UI`显示相关方法和逻辑，来看几个比较重要的方法。

```objc
- (void)setRequest:(LKImageRequest *)request
{
    request.internalProcessorList = self.processorList;
    if (![_request isEqual:request] || _request.error)
    {
        [self.imageManager cancelRequest:_request];
        _request = request;
    }
    if (!_request)
    {
        [self internalSetImage:nil withRequest:nil];
    }

    [self setNeedsLayout];
    if (!self.delayLoadingImage)
    {
        [self layoutIfNeeded];
    }
}

- (void)setLoadingImageRequest:(LKImageRequest *)loadingImageRequest
{
    loadingImageRequest.internalProcessorList = self.processorList;
    if (![_loadingImageRequest isEqual:loadingImageRequest] || _loadingImageRequest.error)
    {
        [self.imageManager cancelRequest:_loadingImageRequest];
        _loadingImageRequest = loadingImageRequest;
    }
    [self setNeedsLayout];
}

- (void)setFailureImageRequest:(LKImageRequest *)failureImageRequest
{
    failureImageRequest.internalProcessorList = self.processorList;
    if (![_failureImageRequest isEqual:failureImageRequest] || _failureImageRequest.error)
    {
        [self.imageManager cancelRequest:_failureImageRequest];
        _failureImageRequest = failureImageRequest;
    }
    [self setNeedsLayout];
}
```
因为三张图片（分别对应`request`,`loadingImageRequest`,`failureImageRequest`）显示在同一个`view`上，所以他们的`processorList`应该相同，才会有相同的效果。设置完`request`之后，都会调用一遍`setNeedsLayout`来触发`layoutSubviews`。`setRequest:`还会根据`delayLoadingImage`变量来判断是否需要调用`layoutIfNeeded`来立刻调用`layoutSubviews`。


再来看看`layoutSubviews`做了什么事情。
```objc
- (void)layoutSubviews
{
    [super layoutSubviews];
    [self layoutAndLoad];
}

- (void)layoutAndLoad
{
    [self layoutImageView];
    if (self.request.state == LKImageRequestStateFinish
        &&!CGSizeEqualToSize(self.oldSize, self.size)
        &&(!self.presentationImage||self.presentationImage.lk_isScaled))
    {
        [self.request reset];
    }

    if (self.request && self.request.state == LKImageRequestStateInit)
    {
        self.oldSize = self.size;

        __weak LKImageView *wself = self;

        
        

        if (!self.request.synchronized)
        {
            if (self.loadingImageRequest)
            {
                [self dealWithRequest:self.loadingImageRequest];
                [self.imageManager sendRequest:self.loadingImageRequest
                                    completion:^(LKImageRequest *request, UIImage *image, BOOL isFromSyncCache) {
                                        [wself handleRequestFinish:request image:image isFromSyncCache:isFromSyncCache];
                                    }];
            }
        }
        
        [self dealWithRequest:self.request];
        [self.imageManager sendRequest:self.request
                            completion:^(LKImageRequest *request, UIImage *image, BOOL isFromSyncCache) {
                                [wself handleRequestFinish:request image:image isFromSyncCache:isFromSyncCache];
                            }];
    }
}
```
`layoutSubviews`调用了`layoutAndLoad`方法，这个方法首先对`UIImageView`重新布局了一次。
接着根据条件判断`self.request`需不需要进行`reset`。
然后判断是否需要加载`request`，最后根据`request`的状态和属性来将`loadingImageRequest`和`request`发送给`imageManager`并设置好完成的回调方法。


回调处理函数有点长，但是其实逻辑还算比较简单。根据不同的`request`进行不同的处理。
```objc
- (void)handleRequestFinish:(LKImageRequest *)request image:(UIImage *)image isFromSyncCache:(BOOL)isFromSyncCache
{
    // dispatch到主线程中处理
    [LKImageUtil async:dispatch_get_main_queue()
                 block:^{
                     if (request == self.request)
                     {
                         // 如果是self.request 不做什么事情
                         LKImageLogVerbose(@"requst finish");
                     }
                     else if (request == self.failureImageRequest)
                     {
                         // 如果是self.failureImageRequest
                         // 并且self.request.state还没完成，或者没有error
                         // 那么现在不需要将self.failureImageRequest的image赋给UIImageView
                         LKImageLogVerbose(@"failureImageRequest finish");
                         if (self.request.state != LKImageRequestStateFinish || !self.request.error)
                         {
                             return;
                         }
                     }
                     else if (request == self.loadingImageRequest)
                     {
                         // 如果是self.loadingImageRequest
                         // 并且self.request已经完成，那么这个请求的结果也就没什么用了。
                         LKImageLogVerbose(@"loadingImageRequest finish");
                         if (self.request.state == LKImageRequestStateFinish)
                         {
                             return;
                         }
                     }
                     else
                     {
                         // 如果都不是，那么只可能是三个request中至少有一个被赋予了新的值
                         // 如果self.request发生了改变，那么有两种情况，
                         // 1. 新的self.request还没执行layoutAndLoad， -> 立刻执行一次
                         // 2. 新的self.request已经执行了layoutAndLoad， -> 只执行layoutImageView
                         // 如果self.request没变 -> 只执行layoutImageView
                         [self layoutAndLoad];
                         return;
                     }

                     // 如果请求成功时，View的大小已经发生了变化，并且image是经过了缩放的
                     // 那么说明image已经被处理为了非原始的尺寸
                     // 将这个request重置，调用layoutAndLoad尝试重新请求。
                     if (!CGSizeEqualToSize(self.oldSize, self.size)&&image.lk_isScaled)
                     {
                         [request reset];
                         [self layoutAndLoad];
                         return;
                     }

                     // 发生了error
                     if (request.error)
                     {
                         // 如果是被取消的，重置后layoutAndLoad一次，重新发送一次请求
                         if (request.error.code == LKImageErrorCodeCancel)
                         {
                             [request reset];
                             [self layoutAndLoad];
                         }
                         else
                         {
                             // 如果是其他原因
                             // 判断是否是failureImageRequest本身
                             // 如果不是，执行failureImageRequest，请求失败图片
                             LKImageLogError([NSString stringWithFormat:@"%@ error:%@", request, request.error.description]);
                             if (![request isEqual:self.failureImageRequest])
                             {
                                 [self dealWithRequest:self.failureImageRequest];
                                 __weak LKImageView *wself = self;
                                 [self.imageManager sendRequest:self.failureImageRequest
                                                     completion:^(LKImageRequest *request, UIImage *image, BOOL isFromSyncCache) {
                                                         [wself handleRequestFinish:request image:image isFromSyncCache:isFromSyncCache];
                                                         
                                                     }];
                             }

                             if ([self.delegate respondsToSelector:@selector(LKImageViewImageDidLoad:request:)])
                             {
                                 [self.delegate LKImageViewImageDidLoad:self request:request];
                             }
                         }
                     }
                     else
                     {
                         // 加载中
                         if (request.progress < 1.0)
                         {
                             if ([self.delegate respondsToSelector:@selector(LKImageViewImageLoading:request:)])
                             {
                                 [self.delegate LKImageViewImageLoading:self request:request];
                             }
                         }

                         if (image)
                         {
                             // 处理动图
                             if (!self.imageView.image && !request.synchronized &&
                                 ((isFromSyncCache && self.fadeMode & LKImageViewFadeModeAfterCached) ||
                                     (!isFromSyncCache && self.fadeMode & LKImageViewFadeModeAfterLoad)))
                             {
                                 [self playFadeAnimation];
                             }

                             // 设置图片
                             [self internalSetImage:image withRequest:request];
                             if (request.progress >= 1.0)
                             {
                                 if ([self.delegate respondsToSelector:@selector(LKImageViewImageDidLoad:request:)])
                                 {
                                     [self.delegate LKImageViewImageDidLoad:self request:request];
                                 }
                             }
                         }
                     }

                 }];
}

```


## Managers和背后真正干活的人
Managers负责管理和执行`request`。`request`的生命周期和这些Manager密切相关。前面已经说明了各个Manager间的关系。
在`LKImageKit`中有这么几个Manager：
+ `LKImageManager`
+ `LKImageDecoderManager`
+ `LKImageCacheManager`
+ `LKImageLoaderManager`
+ `LKImageProcessorManager`

要弄清楚一个`request`是如何获得`Image`的流程，我们就需要分析清楚这几个Manager之间的是如何协作的。


### LKImageManager
让我们先从`LKImageManager`开始分析，这个Manager是`LKImageKit`最为核心的Manager。

我们根据`request`的执行过程来逐一分析相关的变量和方法。
#### 成员变量
```objc
@property (nonatomic) NSMutableDictionary<NSString *, LKImageRequest *> *requestDic;
@property (nonatomic) NSOperationQueue *queue;
```
+ `requestDic`用来存储`requestLV1`。至于这个`LV1`是啥意思，下面会讲到。还有一点需要值得注意，这里面只存储`synchronized`为`NO`的`request`，换句话说，只存`异步`的`request`。
+ `LKImageManager`拥有一个`queue`，它的任务都在这个队列中执行。`queue`的设置在`initWithConfiguration:`方法里可以查看，是一个串行队列。所以说，`LKImageManager`的所有`op`都是按序串行执行的。

#### sendRequest:
这是`request`发起的地方，记录了执行开始的时间，检查是否有`cache`，最后调用了`combineRequest:`方法，这个方法才是关键。
这里要注意的是这个方法的形参`requestLV0`，可能会很奇怪为什么要这样命名，其实`request`后头这个`LV0`非常重要，能够帮助我们理解`request`的组织架构。

#### combineRequest:

继续来看`combineRequest:`方法，抛去`op`的实际运行代码，这个方法做的事情就是将`request`标记为开始，将全局的`LKImageRunningRequestCount`计数加`1`，然后把`op`丢到队列里，准备执行。

重点来看剥离出来的`op`实际需要执行的代码。
```objc
	if (requestLV0.isCanceled)
	{
	    // ... 清理工作
	    return;
	}
	LKImageLogVerbose([NSString stringWithFormat:@"LKImageManagerProcessRequest:%@", requestLV0]);
	LKImageRequest *requestLV1 = nil;
	requestLV0.error           = nil;
	if (!requestLV0.synchronized)
	{
	    requestLV1 = [self.requestDic objectForKey:requestLV0.identifier];
	}

	if (requestLV1)
	{
	    LKImageLogVerbose([NSString stringWithFormat:@"ManagerRequestCombine:%@", requestLV1]);
	    [requestLV1 addChildRequest:requestLV0];
	    return;
	}
	else
	{
	    requestLV1 = [requestLV0 createSuperRequest];
	    if (!requestLV0.synchronized)
	    {
	        LKImageLogVerbose([NSString stringWithFormat:@"ManagerRequestCreate:%@", requestLV1]);
	        [self.requestDic setObject:requestLV1 forKey:requestLV1.identifier];
	    }
	}

	[self loadRequest:requestLV1];
```
执行前先检查`request`是否已被`cancel`了。
如果没有，会构造出一个名为`requestLV1`的`request`，并且将其设置为`requestLV0`的父亲。最后调用`loadRequest:`方法`load`这个`requestLV1`。也就是说，`LKImageManager`实际上处理的`request`是`requestLV1`而不是初始的`requestLV0`，那么为什么要这样做呢？
让我们来仔细看一看。
首先判断了`requestLV0`是否是`synchronized`，如果不是，就尝试从`requestDic`里取，如果已经存在，则直接将`requestLV0`设为它的孩子。
否则的话，使用`requestLV0`构造一个`requestLV1`。
不难看出，`requestLV1`可能对应有多个`requestLV0`，当`requestLV1`完成时，其所有的`requestLV0`都可被视作完成了，这样的结构设计可以避免发送重复的请求。
当然这不会作用于设置了`synchronized`的请求，每个设置了`synchronized`的`requestLV0`都独立拥有一个`requestLV1`。

分析完正常的流畅再来看一眼清理工作的代码，这里我第一次看的时候也有点不明白，建议先看`cancelRequest:`的代码。
```objc
if (requestLV0.isCanceled) {
	atomic_fetch_sub(&LKImageRunningRequestCount, 1);
    atomic_fetch_add(&LKImageCancelRequestCount, 1);
    requestLV0.isFinished = YES;
    requestLV0.error      = [LKImageError errorWithCode:LKImageErrorCodeCancel];
    [requestLV0 managerCallback:nil isFromSyncCache:NO];
    // 1
    [requestLV0.imageManagerCancelOperation cancel];
    requestLV0.imageManagerCancelOperation = nil;
}
```
初看起来没什么大问题，但是注意到`1`这里直接`cancel`调了`imageManagerCancelOperation`，而没有做其他事情，觉得有点奇怪。
主要问题是这么做**是否会造成requestLV1里的requestLV0不能正确移除？**

我们可以来分析一下：
首先代码进入这个`if`的条件是：调用了`combineRequest`之后，在这个`op`开始执行前，调用了`cancelRequest`。
那么因为这个`queue`的并发数为1，是个串行队列。而且`requestLV0`的`priority`和`requestLV1`的一样。那么始终会先执行这个`op`，这时`LV0`还没有被加入到`LV1`中，所以并不需要移除。而且`LKImageRunningRequestCount`是在`op`执行前被`add`的，所以这里也可以直接`sub`。


#### loadRequest:
该方法向`loaderManager`发起了`requestLV1`请求，显然这是需要`loaderManager`去`load`图片了。
`load`的过程这里不关心。回调的工作也被封装成了一个`op`，丢到了自己的`queue`里去跑。

如果`request`发生了错误，调用`requestDidFinished:`做清理工作，调用`managerCallback:isFromSyncCache:`来通知`request`。
如果成功返回了image，调用`processRequest:request`，对image进行最后一项工作。

#### `processRequest:request:`
该方法对加载成功的图片做最后的处理，将`request`丢给`processorManager`，依次执行`request`中的`processorList`里的每个`processor`。	

#### cancelRequest:
```objc
- (void)cancelRequest:(LKImageRequest *)requestLV0
{
    if (!requestLV0.isStarted || requestLV0.isCanceled || requestLV0.isFinished)
    {
        return;
    }
    // 1
    requestLV0.isCanceled = YES;
    [requestLV0.imageManagerCancelOperation cancel];
    lkweakify(requestLV0);

    requestLV0.imageManagerCancelOperation = [NSBlockOperation blockOperationWithBlock:^{
        lkstrongify(requestLV0);
        requestLV0.imageManagerCancelOperation = nil;
        if (!requestLV0.isStarted || requestLV0.isFinished)
        {
            return;
        }
        requestLV0.isFinished = YES;
        LKImageLogVerbose([NSString stringWithFormat:@"LKImageManager Cancel Request:%@", requestLV0]);

        LKImageRequest *requestLV1 = [self.requestDic objectForKey:requestLV0.identifier];
        if (!requestLV1)
        {
            LKImageLogWarning(@"Cancel a invalid request,request not found in manager");
            return;
        }

        requestLV0.error = [LKImageError errorWithCode:LKImageErrorCodeCancel];
        NSUInteger index = [requestLV1.requestList indexOfObject:requestLV0];
        if (index != NSNotFound)
        {
            atomic_fetch_sub(&LKImageRunningRequestCount, 1);
            atomic_fetch_add(&LKImageCancelRequestCount, 1);
            // 2
            [requestLV0 managerCallback:nil isFromSyncCache:NO];
            [requestLV1 removeChildAtIndex:index];
            // 3
            if (requestLV1.requestList.count == 0)
            {
                LKImageLogVerbose([NSString stringWithFormat:@"ManagerTaskDidCanceled:%@", requestLV1.identifier]);
                [self.requestDic removeObjectForKey:requestLV1.identifier];
                [self.loaderManager cancelRequest:requestLV1];
            }
        }
        else
        {
            LKImageLogWarning(@"Cancel a invalid task,request not found in manager");
            return;
        }
    }];
    requestLV0.isCanceled                  = YES;
    [self.queue lk_addOperation:requestLV0.imageManagerCancelOperation request:requestLV0];
}

```

顾名思义，这个方法用来取消正在执行的`request`。
1. 方法将`requestLV0`立刻标记为`已取消`，如果之前有执行的`imageManagerCancelOperation`，停止它，避免重复执行。
2. 在`imageManagerCancelOperation`里，将`requestLV0`从对应的`requestLV1`的孩子中删除。
3. 如果删除后`requestLV1`已经没有孩子了，那么说明它也没有存在的意义了，从`requestDic`中移除，然后向`loaderManager`发起取消`requestLV1`的请求。

#### `requestDidFinished:`
这个方法唯一值得注意的地方就是对`requestLV0`的处理，在`requestLV1`执行完之后，将其自己从`requestDic`中移出，并将所有的孩子`requestLV0`都标记为完成，而且会取消它们的`imageManagerCancelOperation`，也就是说在取消发起后至取消真正开始执行前，如果`request`完成了，那么取消操作将不会被执行。
```objc
- (void)requestDidFinished:(LKImageRequest *)requestLV1
{
    requestLV1.isFinished = YES;
    if (!requestLV1.synchronized)
    {
        [self.requestDic removeObjectForKey:requestLV1.identifier];
    }
    LKImageLogVerbose([NSString stringWithFormat:@"ManagerRequestDidFinish:%@", requestLV1]);
    atomic_fetch_add(&LKImageFinishRequestCount, requestLV1.requestList.count);
    atomic_fetch_sub(&LKImageRunningRequestCount, requestLV1.requestList.count);
    // 清理孩子request
    for (LKImageRequest *requestLV0 in requestLV1.requestList)
    {
        requestLV0.isFinished = YES;
        [requestLV0.imageManagerCancelOperation cancel];
    }
}
```

### LKImageLoaderManager
`LKImageLoaderManager`负责加载图片，这里的加载定义的比较广，加载的方式可以是从网络下载、本地文件加载、相册加载、内存加载等等。
同时，`LKImageLoaderManager` 也有图片解码的功能，在头文件中我们可以看见定义了
```objc
@property (nonatomic, strong) LKImageDecoderManager *decoderManager;
```
因为从不同的地方加载返回的数据类型可能不同，有两种数据类型：
- 压缩格式的数据（`jpg`,`png`,`jpeg`...)
- 解码后的位图数据

#### imageWithRequest:callback:
核心代码就是这个`op`。
```objc
// 构建一个requestLV2，实际是load requestLV2这个request。
// 如果已经有个相同key的loader，则往已经存在的request上添加child
NSOperation *op           = [NSBlockOperation blockOperationWithBlock:^{
    requestLV1.isStarted = YES;
    if (requestLV1.isCanceled)
    {
        requestLV1.isFinished = YES;
        requestLV1.error      = [LKImageError errorWithCode:LKImageErrorCodeCancel];
        [requestLV1 loaderCallback:nil];
        [requestLV1.loaderManagerCancelOperation cancel];
        requestLV1.loaderManagerCancelOperation = nil;
        return;
    }
    // 构建LV2
    LKImageRequest *requestLV2 = nil;
    if (requestLV1.keyForLoader && !requestLV1.synchronized) //没有key的不合并
    {
        requestLV2 = self.requestDic[requestLV1.keyForLoader];
    }
    if (!requestLV2)
    {
        requestLV2          = [requestLV1 createSuperRequest];
        requestLV2.priority = requestLV1.priority;
        // 1
        requestLV2.loader   = [self loaderForRequest:requestLV1];
        if (!requestLV2.loader)
        {
            requestLV2.error = [LKImageError errorWithDomain:LKImageErrorDomain code:LKImageErrorCodeLoaderNotFound userInfo:nil];
            [requestLV2 loaderCallback:nil];
            return;
        }

        if (requestLV1.keyForLoader && !requestLV1.synchronized) //没有key的不合并
        {
            [self.requestDic setObject:requestLV2 forKey:requestLV1.keyForLoader];
        }

        [self loadRequest:requestLV2];
    }
    else
    {

        [requestLV2 addChildRequest:requestLV1];
    }
}];
```
和`LKImageManager`一样，构造了`requestLV2`，来有效的避免重复的请求。
1. 在构造`requestLV2`的时候多做了一件事件，为`LV2`寻找合适的`loader`。在`loaderList`里寻找能加载当前`request`的`loader`


#### loadRequest:
根据`loader`相应的方法不同，执行不同的代码，一类`loader`加载完成后得到的就是`Image`对象，这类`loader`加载完成后不需要解码操作；另一类`loader`加载完成后得到的是`Data`对象，需要进一步的解码操作才能获得`Image`对象。分别交给`loadImageRequestFinished:image:progress:error:`和`loadDataRequestFinished:data:progress:error:`处理。
```objc
lkweakify(self);
requestLV2.loaderOperation = [NSBlockOperation blockOperationWithBlock:^{
    lkstrongify(self);
    if (requestLV2.isCanceled)
    {
        return;
    }
    
    // ? 看起来都是同步执行的，为什么要用信号量？
    LKDispatch(requestLV2.synchronized, requestLV2.loader.gcd_queue, ^{
        if (requestLV2.isCanceled)
        {
            dispatch_semaphore_signal(requestLV2.loader.semaphore);
            return;
        }
        requestLV2.isStarted = YES;
        [requestLV2.loader dataWithRequest:requestLV2
                                  callback:^(LKImageRequest *requestLV2, NSData *data, float progress, NSError *error) {
                                      // ... load Image or load Data finished operation
                                  }];
    });
    if (!requestLV2.synchronized)
    {
        dispatch_semaphore_wait(requestLV2.loader.semaphore, DISPATCH_TIME_FOREVER);
    }
    requestLV2.loaderOperation = nil;
}];
[requestLV2.loader.queue lk_addOperation:requestLV2.loaderOperation request:requestLV2];
```
需要注意的是`loaderOperation`是在`requestLV2.loader.queue`里执行的。这个`queue`是个并行队列，每个`loaderOperation`应该占据一个并发线程，所以在`loaderOperation`执行完成之前，使用了信号量来阻塞住线程，来实现同时至多只有`n`个`op`并发。

#### loadImageRequestFinished:image:progress:error:
`image`类型加载完成的回调，因为是加载返回的直接是`UIImage`类型的数据，不需要解码，调用`requestLV2`的`loaderCallback:`通知图片加载完成。

#### loadDataRequestFinished:data:progress:error:
`data`类型的加载完成回调，因为加载完成返回的结果是`NSData`类型的数据，还需要交给`decoder`进行解码才能获得`UIImage`类型的图片。之后才会调用`requestLV2`的`loaderCallback:`。

### LKImageDecoderManager
这个Manager的工作很简单。它本身持有一个`decoderList`，对外接口最重要的是
```objc
- (void)decodeImageFromData:(NSData *_Nullable)data request:(LKImageRequest * _Nullable)request complete:(LKImageDecoderCallback)complete;
```
调用这个方法后会按序使用`decoderList`里每一个`decoder`尝试去解码这份`data`直到有一个`decoder`能解码成功或者试完了所有`decoder`为止。

### LKImageProcessorManager
图片加载完成后可能需要进行处理，`LKImageProcessorManager`就是管理这些处理操作的，`request`拥有一个`processorList`，这个`List`表明了这个图片需要进行哪些处理操作，`LKImageProcessorManager`对图片逐一的进行处理操作。

### LKImageCacheManager
cacheManager也被设计成了拥有一个`cacheList`的形势，可以自定义`cache`，实现自己的`cache`策略。

```objc
- (UIImage *)imageForRequest:(LKImageRequest *)request continueLoad:(BOOL *)continueLoad;
- (void)cacheImage:(UIImage *)image forRequest:(LKImageRequest *)request;

```
这两个方法分别用来读取cache和更新cache。


至此，所有的Manager都分析完了，这有助于我们理解`LKImageKit`的框架和设计思想。但是这些Manager并不做具体的工作，而是将任务派发下去，想要了解具体每个功能是如何实现的，还需要分析`Components`目录下的类，这些才是最底下做事情的类。

下面是`LKImageKit`的UML类图
![Class Diagram](LKImageKit-Class-Diagram.png)