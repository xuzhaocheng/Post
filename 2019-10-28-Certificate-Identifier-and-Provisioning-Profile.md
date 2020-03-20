---
title: 'Certificate, Identifier and Provisioning Profile'
date: 2019-10-28 16:40:04
categories:
tags:
    - iOS
---
初接触iOS开发的程序员大概都被这三样东西给搞晕过，总是搞不清楚各自的用途，我也一样。
趁有空来整理整理这三者的作用是什么，它们之间又有什么关系。
<!-- more -->
## Identitier
让我们从最容易区分的`identifier`开始，它其实包括两大部分：`Bundle ID`和`App ID`。

### Bundle ID
`Bundle ID`是表示App的唯一标识符，任何两个不同的App不能拥有相同的`Bundle ID`。它通常是以反向域名表示法来命名的，它可能长这样`com.yourcompany.app`。

### App ID
很多人可能会把`App ID`和`Bundle ID`搞混，认为他们是一个东西，其实不然。一个`App ID`可能表示一个或者多个App。
`App ID`是由`Team ID`和`Bundle ID`两部分组成的，如果你的`Team ID`是`TID13939`，而你的`Bundle ID`是`com.yourcompany.app`，那么你的`App ID`就是`TID13939.com.yourcompany.app`。
`App ID`又分为`Explicit App ID`和`Wildcard App ID`两种。顾名思义，一个`Explicit App ID`只能表示一个App，而一个`Wildcard App ID`则可以表示多个App。
`Wildcard App ID`可能长这样`TID13939.com.yourcompany.*`。那么为什么要有`Wildcard App ID`呢？那就得知道`App ID`它的作用了。

### Capabilities
每个`App ID`有着对应的`Capability List`，它决定了这个`App ID`对应的App（们）有权限使用哪些系统服务（比如我们常见的地理位置服务，内购，推送...都属于这个范畴）。这些设置会被存放在`entitlement`文件中。如果我们的一系列App都拥有相同的`Capability`，使用`Wildcard App ID`就可以避免创建多个`Explicit App ID`。

## Certification
证书的作用是用来验证数字签名的，数字签名和数字证书能保证我们的App是由可信的实体编写和发布的，它没有被其他第三方修改过。签名和证书都依赖于非对称加密，RSA就是一种非常常见的非对称加密算法。这儿有必要介绍一下数字证书和数字签名的工作原理。

数字签名的过程就是使用自己的私钥对需要签名的实体加密，然后将实体和加密过后的数据放在一起交由接收者。在App发布的过程中是先选定一种hash算法对App计算hash值，然后使用私钥对hash值进行加密，再将加密后的hash值和App打包一起发布。Mach-O文件中有专门存放数字签名的部分。

在非对称加密体系中，公钥是可以被任何人知晓的，它存在于数字证书中，在iOS系统打开App前会对App进行验证，使用公钥对签名时候加密的hash值解密，再使用相同的hash算法对App计算hash值，最后比对hash值是否相同，达到验证App是否是由期望的实体发布的目的。但是仅仅这样是不能保证我们的App是安全无害的，任何一个人都能生成一对公私钥，然后发布其公钥，验证的流程一样可以通过。所以我们需要对发布公钥的实体进行验证，这也就是为什么我们需要在Apple Developer Center上上传`CertificateSigningRequest`并下载证书了，Apple在这里充当了一个可信权威第三方实体的角色，Apple使用它的私钥对我们的`CertificateSigningRequest`进行了加密，从而保证了我们的公钥是可信的。

<img src="diagram.002.png" alt="Certification" width="70%" height="70%">

其中申请证书包含了1、2两步：
1. 本地生成`CertificateSigningRequest` -> 生成本地的公私钥对
2. 上传Apple Developer Center，下载证书 -> 上传本地公钥交由Apple使用Apple的私钥签名生成证书并在本地导入证书，将本地私钥与证书关联

如此一来，我们就有了用来签名的私钥和一份用来验证签名的证书。我们就可以用这个私钥对App进行签名。
打包App包含了其中的3：
3. 使用本地的私钥对App进行签名。

> 因为App不止包含Mach-O文件，还包括资源文件和库文件，理论上来说这些文件都需要被签名，才能保证App的完整性。在Mach-O文件中，有一段专门定义的空间从来存储数字签名。其他文件的签名存储在`_CodeSignature/CodeResources`下，这是一个`XML`格式的文件，记录了文件名和其签名。除此之外，库文件也有其单独的签名文件`CodeResources`。

iOS系统会在App启动前对App的签名进行验证，也就是上图中的4、5：
4. 使用Apple的公钥对App中的证书进行验证。
5. 验证通过后使用认证过的公钥对App内的签名进行验证。  

## Provisioning Profile
但是这还不够，除了证书之外，最后的安装包还会包含`entitlement`，`App ID`，如果是Development的安装包，那么还会包含`Device ID`s。Apple把这些文件的集合称之为`Provisioning Profile`。它最后会以`embeded.mobileprovision`存在于`IPA`文件内。
<img src="image.png" alt="Provisioning Profile">

我们可以使用命令
```bash
security cms -D -i embedded.mobileprovision
```
来查看`embedded.mobileprovision`文件中的内容。

`Provisioning Profile`的工作原理如下图所示：
<img src="diagram.003.png" alt="Distribution Certification" width="70%" height="70%">

1. 我们在本地申请`Certificate Signing Request`并把*Pub<sub>m</sub>*上传给Apple。
2. Apple使用*Pri<sub>A</sub>*签名*Pub<sub>m</sub>*，获得一份证书*Cert*，这份证书包含*Signature(Pub<sub>m</sub>)*和*Pub<sub>m</sub>*。
3. Apple使用*Pri<sub>A</sub>*签名*Cert*，*Entitlement*，*App ID*以及*Device ID*s，获得Provisioning Profile*PP*。
4. 使用*Pri<sub>m</sub>*签名App获得*Signature(App)*，连同*PP*和App一起打包成IPA文件。
5. Device上打开App时，系统使用*Pub<sub>A</sub>*验证*PP*，验证成功后再验证*PP*里的证书*Cert*。
6. 使用证书中的*Pub<sub>m</sub>*验证*Signature(App)*。

## App Store上的App
如果你是从App Store上下载的App，打开包你会发现根本找不到`embedded.mobileprovision`文件。这是因为由App Store分发的App不再需要使用生成的证书来验证。我们所有在Apple Store上架的App，都需要将自己的App上传，所以App Store可以使用Apple的私钥去对Apple签名，而不再需要使用*Pub<sub>m</sub>*。而且App Store会对Mach-O文件进行加密，我们本地签的名也会失效。验证的过程就更加简单了：
<img src="diagram.001.png" alt="Distribution Certification">

[这里](https://wereadteam.github.io/2017/03/13/Signature/)有一些更详细的说明。
> 而 AppStore 的签名验证方式有些不一样，前面我们说到最简单的签名方式，苹果在后台直接用私钥签名 App 就可以了，实际上苹果确实是这样做的，如果去下载一个 AppStore 的安装包，会发现它里面是没有 embedded.mobileprovision 文件的，也就是它安装和启动的流程是不依赖这个文件，验证流程也就跟上述几种类型不一样了。

> 据猜测，因为上传到 AppStore 的包苹果会重新对内容加密，原来的本地私钥签名就没有用了，需要重新签名，从 AppStore 下载的包苹果也并不打算控制它的有效期，不需要内置一个 embedded.mobileprovision 去做校验，直接在苹果用后台的私钥重新签名，iOS 安装时用本地公钥验证 App 签名就可以了。

> 那为什么发布 AppStore 的包还是要跟开发版一样搞各种证书和 Provisioning Profile？猜测因为苹果想做统一管理，Provisioning Profile 里包含一些权限控制，AppID 的检验等，苹果不想在上传 AppStore 包时重新用另一种协议做一遍这些验证，就不如统一把这部分放在 Provisioning Profile 里，上传 AppStore 时只要用同样的流程验证这个 Provisioning Profile 是否合法就可以了。

> 所以 App 上传到 AppStore 后，就跟你的 证书 / Provisioning Profile 都没有关系了，无论他们是否过期或被废除，都不会影响 AppStore 上的安装包。


## Reference
[iOS App 签名的原理](https://wereadteam.github.io/2017/03/13/Signature/)
[逆向Appstore应用（二）](https://www.jianshu.com/p/f42073b4bea1)
[iOS - How to Create a Provisioning Profile](https://customersupport.doubledutch.me/hc/en-us/articles/229496268-iOS-How-to-Create-a-Provisioning-Profile)
[Code Signing Guide](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/AboutCS/AboutCS.html)