---
title: 谈谈dispatch_once
date: 2019-10-10 11:46:19
categories: Tech
tags: 
    - iOS
    - GCD
---

相信每个接触过iOS开发的程序员都看见过`dispatch_once`这个方法。在`Objective-C`中实现一个线程安全的单例只需要以下一段代码：
```Objc
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    //... 初始化单例
});
```
仅仅两三行代码，就能实现一个线程安全的单例，而且还有着非常不错的性能。但是为什么如此简单的代码就能实现线程安全的单例呢？`dispatch_once`背后究竟是怎么实现的，又有哪些弊端呢？
<!-- more -->
## 0x00 揭开面纱
略微知道一些`iOS`开发的程序员大概都知道`dispatch_xxx`一类的方法都是由Grand Central Dispatch（以下简称GCD）提供的。它对繁琐的线程操作进行了抽象和封装，使得我们可以更加专注于代码本身，而不会陷入到多线程的困境中，极大的提高了`iOS`程序员的编码效率。Apple公司在2009年9月公开了GCD的源码，现在它有一个名字`libdispatch`。得益于开源，我们可以很方便的找到它的源码来一探究竟。

`libdispatch`经过多年的发展，其代码也经历了诸多的变化，实现上也会有些许不同。`dispatch_once`的源码已经和我第一次看的时候有了很大的不同，这次我们就来分析最新的源码。

以下的源码均来自于`libdispatch-1008.200.78`。[这里](https://opensource.apple.com/tarballs/libdispatch/)可以找到所有历史版本的源码。或者你也可以直接访问[Git仓库](https://github.com/apple/swift-corelibs-libdispatch)来获取最新的源码。
我们这次所要分析的`dispatch_once`函数位于`src/shims/once.c`文件中。

## 0x01 准备知识
粗看GCD的源码，可能会被各种宏定义给搞晕头脑。在分析源码前，先来看一下这次会遇到的几个宏定义：
- os_atomic_xchg(p, v, m)
- os_atomic_cmpxchg(p, e, v, m) 
- os_atomic_cmpxchgvw(p, e, v, g, m)
- os_atomic_load(p, m)
- likely, unlikely

### os_atomic_xchg
原子交换操作。`p`通常是指针类型。
用`v`的值替换掉`*p`的值，并返回`*p`交换前的值。

### os_atomic_cmpxchg
原子比较交换操作。`p`通常是指针类型。
比较`*p`和`e`是否相等，如果相等，则将`*p`的值设为`v`，并且返回`true`。否则不做什么并返回`false`。

### os_atomic_cmpxchgvw
原子比较交换操作。`p`和`g`通常是指针类型。
比较`*p`和`e`是否相等，如果相等，则将`*p`的值设为`v`，并返回`true`。否则将`*p`的值赋给`*g`，并返回`false`

### os_atomic_load
原子加载操作。`p`通常是指针类型。
将`*p`的值取出来并返回。

### likely unlikely
这两个宏能帮助编译器做分支预测([Branch Prediction](https://en.wikipedia.org/wiki/Branch_predictor))。
看一个例子：
```C
if (likely(a == b)) {
    // do something
}
```
就表示告诉编译器`a == b`为`true`的可能性更大，编译器就能知道`do something`被执行的概率更大，以此来帮助编译器优化指令。

## 0x02 实战分析
在`once.c`源文件中，我们可以很容易的找到`dispatch_once`函数的定义：
```C
void
dispatch_once(dispatch_once_t *val, dispatch_block_t block)
{
	dispatch_once_f(val, block, _dispatch_Block_invoke(block));
}
```
这里的`dispatch_once_t`，也就是我们上文`Objective-C`代码中传入的`onceToken`，查找定义我们可以发现，这个类型实际上只是一个`long`类型，是不是有点惊讶。

再往下看，`dispatch_once_f`函数。
需要注意的是，这里只保留了最基本的代码。因为现在版本的`libdispatch`有很多配置宏，这里使用的是默认的配置，为了方便阅读，删除了部分无关的代码。

```C
void
dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func)
{
	dispatch_once_gate_t l = (dispatch_once_gate_t)val;
	if (_dispatch_once_gate_tryenter(l)) {
		return _dispatch_once_callout(l, ctxt, func);
	}
	return _dispatch_once_wait(l);
}
```
初看这个函数，也能够很容易猜到它做了什么。
在多线程环境中，第一个进入此函数的线程会进入`if`语句，并且可以执行对应的`block`。在它之后的其他线程则会进入等待阶段。

`dispatch_once_gate_t`是一个联合体，它的定义如下：
```C
typedef struct dispatch_once_gate_s {
	union {
		dispatch_gate_s dgo_gate;
		uintptr_t dgo_once;
	};
}
```
`dispatch_gate_s`里实际上也只有一个`uint32_t`类型的变量。

### 首次进入的线程
我们尽量按时间顺序来分析，假设当前线程是第一个调用`dispatch_once`的线程，那么它的执行流程如下。

首先它会请求进入`_dispatch_once_gate_tryenter`，这里其实就是为了解决资源竞争的问题，用了原子操作而不是加锁的方式。
```C
static inline bool
_dispatch_once_gate_tryenter(dispatch_once_gate_t l)
{
	return os_atomic_cmpxchg(&l->dgo_once, DLOCK_ONCE_UNLOCKED,
			(uintptr_t)_dispatch_lock_value_for_self(), relaxed);
}
```
根据上面的准备知识，我们可以知道，这里利用`dgo_once`的值来判断当前的`token`有没有被执行过的，如果是`DLOCK_ONCE_UNLOCKED`状态，则表示这个`token`没有被执行过，原子操作会赋一个新值给`dgo_once`。一旦新值被赋上，后进入这个函数的线程都会得到`false`的返回值。从而保证了我们`dispatch_once`传入的`block`只会执行一次。


在来看`_dispatch_once_callout`
```C
static void
_dispatch_once_callout(dispatch_once_gate_t l, void *ctxt, dispatch_function_t func)
{
	_dispatch_client_callout(ctxt, func);
	_dispatch_once_gate_broadcast(l);
}
```
`_dispatch_client_callout`就是调用了这个`block`，不多解释。我们把重点放在`_dispatch_once_gate_broadcast`上，从名字上来看，是为了通知其他等待的线程。那么究竟是不是和我们想的一样呢？往下看：

```C
static inline void
_dispatch_once_gate_broadcast(dispatch_once_gate_t l)
{
	dispatch_lock value_self = _dispatch_lock_value_for_self();
	uintptr_t v;

	v = _dispatch_once_mark_done(l);

	if (likely((dispatch_lock)v == value_self)) return;
	_dispatch_gate_broadcast_slow(&l->dgo_gate, (dispatch_lock)v);
}
```
这里`dispatch_lock`实际上是`uint32_t`的类型。**`value_self`是当前线程的`tid`和一个掩码`DLOCK_OWNER_MASK`的按位`与`的结果，也就是说每个线程有其独一无二的`value_self`**。

```C
static inline uintptr_t
_dispatch_once_mark_done(dispatch_once_gate_t dgo)
{
	return os_atomic_xchg(&dgo->dgo_once, DLOCK_ONCE_DONE, release);
}
```
这里可以看到又对`dgo_once`的值做了一次修改，将其赋值为`DLOCK_ONCE_DONE`。并且返回之前的值。这个值很大可能就是前面`_dispatch_once_gate_tryenter`中赋上的值，也就是`value_self`。如果不是，则可能会引起Crash，在`_dispatch_gate_broadcast_slow`中有相应的Error Handling。

简单来说，`dgo_once`的值经历了`DLOCK_ONCE_UNLOCKED`(初始状态) -> `value_self` -> `DLOCK_ONCE_DONE`(执行完毕)三个状态。利用`dgo_once`来解决多线程资源竞争的问题。每次状态的改变都用了原子操作，避免了锁的开销。

### 等待的线程们
让我们再来看看进入等待状态的线程都做了哪些事情。
为了方便阅读，这里已经把宏`os_atomic_rmw_loop`做了展开。

```C
void
_dispatch_once_wait(dispatch_once_gate_t dgo)
{
	dispatch_lock self = _dispatch_lock_value_for_self();
	uintptr_t old_v, new_v;
	dispatch_lock *lock = &dgo->dgo_gate.dgl_lock;
	uint32_t timeout = 1;

	for (;;) {
        /// os_atomic_rmw_loop(&dgo->dgo_once, old_v, new_v, relaxed, ...) begin
        
        bool _result = false; 
        typeof(&dgo->dgo_once) _p = (&dgo->dgo_once); 
        old_v = os_atomic_load(_p, relaxed); 
        do { 
            if (old_v == DLOCK_ONCE_DONE) {
                return; 
            } 
            
            new_v = old_v | (uintptr_t)DLOCK_WAITERS_BIT; 
            if (new_v == old_v) (
                break;
            )
            _result = os_atomic_cmpxchgvw(_p, old_v, new_v, &old_v, relaxed); 
        } while (unlikely(!_result));
        
        /// os_atomic_rmw_loop(&dgo->dgo_once, old_v, new_v, relaxed, ...) end

		if (unlikely(_dispatch_lock_is_locked_by((dispatch_lock)old_v, self))) {
			DISPATCH_CLIENT_CRASH(0, "trying to lock recursively");
		}

		_dispatch_thread_switch(new_v, 0, timeout++);

		(void)timeout;
	}
}
```

这是一个大循环，那么大体我们也能知道这些线程们会在这个循环里一直等待，直到第一个获取到资源的线程执行完所有代码。

根据代码我们可以发现等待的几个条件（`do while`为内循环，`for(;;)`为外循环）:
- 一旦发现`dgo_once`的值变为了`DLOCK_ONCE_DONE`，也就意味着获取到资源的线程已经执行完所有代码了，此时不必再继续等待，直接返回。
- 如果发现`_p`和`old_v`不一致（也就是说明在一次内循环内`dgo_once`的状态发生了变化），则继续循环检测，因为此时的状态可能已经是`DLOCK_ONCE_DONE`，从而使得当前进程有机会退出等待。
- 当一次内循环内`dgo_once`的值没有发生变化，或者已经是在等待状态，那么会终止该次内循环。进而通知系统可以把当前线程的资源让出来给其他线程使用。

值得注意的是: 外循环内有一个`if`语句来判断是否发生了异常。理想状态下，当前线程的`value_self`与`dgo_once`的值不应该相等，因为一旦进入了等待状态，那么意味着有另外一个线程已经获取了资源，`dgo_once`的值也应该为获取到资源的线程的`value_self`。

*`_dispatch_thread_switch`*底层调用的是`thread_switch`。


## 一些思考
`dispatch_once`的实现使用了若干原子操作来规避了锁的使用，从而以非常小的开销实现了线程安全。但是这种实现也并非完美，再某些情况下可能会造成意想不到的问题。

我们来看这样一个例子:
```Objc
@implementation SingletonA

+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    static SingletonA *_instance = NULL;
    dispatch_once(&onceToken, ^{
        _instance = [[SingletonA alloc] init];

    });
    return _instance;
}

- (instancetype)init {
    self = [super init];
   if (self) {
       [SingletonB sharedInstance];
   }
   return self;
}

@end

```

```Objc
@implementation SingletonB
+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    static SingletonB *_instance;
    dispatch_once(&onceToken, ^{
        _instance = [[SingletonB alloc] init];
    });

    return _instance;
}

- (instancetype)init {
    self = [super init];
   if (self) {
       [SingletonA sharedInstance];
   }
   return self;
}
@end
```

只要初始化任何一个单例，程序就会Crash。分析完`dispatch_once`的源码后，我们可以很快的知道原因。
让我们来复盘一个这个场景:
假设我们调用了`[SingletonA sharedInstance];`。正常来说`SingletonA`的`dispatch_once`会被调用，此时`onceToken`的值被修改为`value_self`。在这个`block`中，我们又调用了`[SingletonB sharedInstance];`，很不幸，`SingletonB`再次调用了`[SingletonA sharedInstance]`。此时`_dispatch_once_gate_tryenter`会失败，因为`onceToken`的值不再是`DLOCK_ONCE_UNLOCKED`了，因此进入了循环等待阶段。但是它永远也等不来`DLOCK_ONCE_DONE`了，它被它自己给阻塞了，于是这个线程就被卡死了。

另外值得注意的一点是，从源码分析来看，`dispatch_once`正确执行的关键是我们传入的`onceToken`。它本质上是一个`long`类型的参数，并且初始化的值必须为`0l`（`DLOCK_ONCE_UNLOCKED`的实际值就是`0l`）。显然，如果我们在外部对`onceToken`进行修改，将会影响到`dispatch_once`执行的结果。比如在单例生成完之后，我们手动将`onceToken`的值赋值为0，那么再次调用则会重新生成一个新的实例。所以我们必须非常小心的对待`onceToken`。当然，我们也可以利用这一特点，来实现的功能，比如有时候我们需要将单例初始化，一个暴力的改法就是重置`onceToken`的值。


## Reference 
https://en.wikipedia.org/wiki/Branch_predictor
https://www.cnblogs.com/haippy/p/3306625.html
http://web.mit.edu/darwin/src/modules/xnu/osfmk/man/thread_switch.html
https://github.com/apple/swift-corelibs-libdispatch
http://lingyuncxb.com/2018/02/01/GCD%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902%20%E2%80%94%E2%80%94%20dispatch-once%E7%AF%87/