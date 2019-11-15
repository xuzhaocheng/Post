---
title: iOS Static Library and Dynamic Library
date: 2019-11-13 14:36:35
categories: Tech
tags:
    - iOS
---

iOS的Library可以分为Static和Dynamic两种。在iOS 8以前，Apple是不允许iOS应用使用Dynamic Library的。常见的静态库有`.a`格式的静态库和`.framework`格式的静态库，其中`.a`格式的静态库是一个单独的二进制文件，是不包含`.h`头文件的，使用时需结合`.h`文件使用。而`.framework`的静态库文件可以看做是`.a` + `.h` + `resource` + `codeSign`，它可以直接拿过来用，使用起来比`.a`要方便许多。`.framework`的动态库文件可以看做是`dylib` + `.h` + `resource` + `codeSign`。

## .a静态库
制作`.a`静态库的方法网上已经有很多了，需要注意的几个地方：
- 可以设置一个总的`.h`头文件，包含所有对外接口。
- 需要暴露出来的`.h`文件需要加到`Build Phases` -> `Copy Files`下，这样导出的`include`下才有这些`.h`文件。
- 目标项目使用`.a`文件时，需要在`Build Setting` -> `Header Search Paths`里添加`.h`文件的搜索路径。
- 注意生成的二进制文件对应的设备架构，如需要合并不同架构的`.a`文件，参考`lipo`命令。
- 如需支持`bitcode`，需要在`Build Settings` -> `Other C Flags`和`Other Linker Flags`里添加`-fembed-bitcode`参数。
- 如果有资源文件，使用`Bundle`来打包它们。

## .framework静态库
`.framework`静态库制作方法比起`.a`静态库要方便许多，使用起来也较为方便，需要注意的点：
- Xcode的`framework`模板创建的工程默认是Dynamic的，需要将`Build Settings` -> `Mach-O Type`设置为`Static Library`
- 做好内外分离，暴露尽可能少的接口，具体可以在`Build Phases` -> `Headers`下配置。
- 因为`.framework`其实只是`.a`和其他文件的打包，也需要注意生成的二进制文件对应的设备架构，如需要合并不同架构的`.a`文件，参考`lipo`命令。
- 如需支持`bitcode`，需要在`Build Settings` -> `Other C Flags`和`Other Linker Flags`里添加`-fembed-bitcode`参数。

## .framework动态库
`.framework`动态库的制作是最简单的，需要注意以下几点：
- 做好内外分离，暴露尽可能少的接口，具体可以在`Build Phases` -> `Headers`下配置。
- 集成`.framework`动态库的时候，需要在`General` -> `Framework, Libraries, and Embedded Content`里将对应的动态库设置为`Embed & (without) Sign`。否则在运行时会抛一个`Reason: image not found`错误。因为iOS系统并没有你的这个Framework，需要嵌入到你的App里。
- 如需支持`bitcode`，需要在`Build Settings` -> `Other C Flags`和`Other Linker Flags`里添加`-fembed-bitcode`参数。
- 动态库也少不了要合并的时候，参考`lipo`命令。

### 签名
动态库的签名必须和App的签名有着相同的`TeamIdentifier`，否则加载时会抛出`code signature invalid`的错误。如果在工程配置里选择了`Embed & Sign`，则一般不会出现这个问题，因为动态库会被重签名一次。关于签名，可以使用`codesign -dv`来查看动态库和App的签名信息。

### 动态库热更新
iOS刚支持动态库的时候就有不少人打过类似的主意。首先我们需要知道，动态库的加载有两种方式：
- App启动时让系统为我们自动加载
- App运行时按需加载，按需卸载

如果想用第一种方法加载动态库，我们只需要在`General` -> `Framework, Libraries, and Embedded Content`里加进需要的动态库并设置为`Embed & (without) Sign`。使用
```bash
otool -l xxx.app/[可执行文件]
```
命令看查看会发现其中有一条加载动态库的`Load Command`，类似这样
```
Load command 14
    cmd LC_LOAD_DYLIB
    cmdsize 80
    name @rpath/DynamicFramework.framework/DynamicFramework (offset 24)
```
这就说明在App启动时会去加载这个动态库。

如果想用第二种方法加载动态库，我们只需要在前面的基础上，去`Build Phases` -> `Link Binary With Libraries`里移除（但是要保留其在`General` -> `Frameworks, Libraries, and Embedded Content`），这样在App启动时，就不会自动去加载动态库。相应的，和之前的可执行文件对比，你会发现可执行文件中少了加载对应动态库的`Load Command`。而且这样做以后，只能使用`Runtime`的方式去调用动态库中的类。

```objc
#import "ViewController.h"
#import <StaticFramework/StaticFramework.h>
#include <dlfcn.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    StaticFrameworkObject *sfobj = [[StaticFrameworkObject alloc] init];
    [sfobj say];

    // DynamicFrameworkObject *dfobj = [[DynamicFrameworkObject alloc] init];
    // [dfobj say];

    void *lib = dlopen("DynamicFramework.framework/DynamicFramework", RTLD_LAZY);

    if (lib) {
        Class dfobjCls = NSClassFromString(@"DynamicFrameworkObject");
        if (dfobjCls) {
            id obj = [[dfobjCls alloc] init];
            if ([obj respondsToSelector:@selector(say)]) {
                [obj performSelector:@selector(say)];
            }
        }
        dlclose(lib);
    } else {
        printf("%s\n", dlerror());
    }
}

@end

```
似乎这样看起来就完美了，我们可以通过替换动态库，然后动态加载新的动态库来实现热更新的功能。思考一下，如果我们要替换动态库，新的动态库就只有放在沙盒下。这在iOS 10之前都是没问题的，但是在iOS 10上（真机），iOS禁止了App从沙盒加载动态库的权限，你会发现类似这样的错误：`file system sandbox blocked mmap()`。但是在iOS 13（真机）上，却又发现能够从沙盒加载动态库了。经过测试，发现在iOS 10，iOS 12（没有iOS 11的设备）上不能从沙盒加载动态库，但是在iOS 13上可以从沙盒加载动态库。不知道是出于什么原因，Apple在iOS 13上又开放了这一权限。

即使抛开从沙盒加载动态库的权限不谈，上架App Store的App是不被允许下载可执行代码的，所以这对于通过App Store分发的App来说并不可用。但是通过in-house和ad-hoc方式分发的App则没有这个约束，完全可以使用。事实上某些企业应用也是这样做的。

这些限制都是对于真机而言的，模拟器来说没有这些限制。相信每一位iOS开发者都被又臭又长的Xcode编译时间恶心过，修改一行码，编译十分钟，是常有的事情。我们完全可以利用动态库的特点来优化。譬如把App拆成多个Framework，修改代码后只用重编一个Framework，再用上动态加载的技术，就能够不重启App来查看最新的修改了。

## 静态库 VS 动态库
首先我们必须知道iOS上的动态库和macOS上的动态库有着些许差别：非系统的动态库并不能被share。也就是说，如果你有两个App都使用了同一个动态库，其实上它们用的是“两份”动态库，虽然它们的内容一模一样。那么Apple为什么要在iOS上支持使用非系统的动态库呢？答案是为了支持`App Extension`。
那么share library这一优势在iOS上不存在了，我们在iOS上使用动态库是不是就没有意义了呢？似乎至少现在看起来是这样的。让我们从几个方面来分别看一看。

### 包体大小
我们知道静态库与动态库的一个区别是：最后的可执行文件会包括其使用到的静态库中的目标文件。静态库实际上是`.o`目标文件的集合，在静态链接的过程中，会把未定义（undefined）的符号（symbol）找到，把未知的地址填上，然后将对应的`.o`文件合并，这会增大我们的包体体积，如果有n个App都用到了同一个静态库，那么就会有n份相同的静态库。但是在iOS下非系统的动态库也有这个问题，虽然不会被添加到可执行文件中，但是它仍会存在于`.ipa`中（`xxx.app/Frameworks/`）。 

### 启动时间
动态库的启动时间肯定是比静态库要慢的，因为它把链接的事情挪到运行时来做了。而且如果一个App加载的动态库数量过多，还会对启动时间造成比较大的影响。一个解决方案是将使用到的动态库合并，将加载多次动态库变为加载一次。如果具体想知道App启动时加载了哪些动态库，可以使用`otool`命令查看可执行文件。
```bash
otool -l [可执行二进制文件]
```
找到`LC_LOAD_DYLIB`类型的`Load Command`，就是App启动时需要加载的动态库了。
Apple声称iOS 13中的`Dynamic Linker Loader 3`(`dydl3`)也支持了对非系统动态库加载的优化，为其提供缓存机制，加速启动过程。具体可以参阅[WWDC2019 Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423/)。

### 内存
因为iOS并不能在进程间共享非系统的动态库，所以内存上的优势比较有限。但是动态库相比于静态库的一个优势是可以按需加载（使用`dlopen()`或者`[NSBundle loadAndReturnError:]`），在不使用的时候还可以卸载它们（`dlclose()`或者`[NSBundle unload]`）。这样做当然能够减少内存的使用量。但是用这种办法加载的动态库，并不会被`dydl3`缓存。

### 开发
使用动态库相比起静态库而言，在开发阶段带给我们的便利还是挺多的。每次`Build`时，如果动态库的内容没有发生变化，我们不必编译整个App，只需要编译宿主App部分的代码，模块化的App能提高编译效率，也更利于团队协作。


## Reference 
[iOS动态库制作以及遇到的坑](https://www.jianshu.com/p/87a412b07d5d)
[iOS静态库 【.a 和framework】【超详细】](https://my.oschina.net/kaqijiang/blog/649632)
[iOS静态库与动态库的区别与打包](https://juejin.im/post/5dc92e92f265da4d4d0d0054#heading-2)
[彻底理解链接器：三，库与可执行文件](https://segmentfault.com/a/1190000016433897)
[iOS 开发中的『库』(二)](https://github.com/Damonvvong/DevNotes/blob/master/Notes/framework2.md)
[Dynamic Library Programming Topics](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/OverviewOfDynamicLibraries.html)