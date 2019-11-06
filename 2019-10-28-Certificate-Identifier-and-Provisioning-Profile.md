---
title: 'Certification, Identifier and Provisioning Profile'
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
证书，这大概是最让开发者头疼的东西之一。首先要明白证书是给代码签名用的，通俗一点，只要签了名，就表示这个App是由谁谁谁编写的，出了事是要找上门的。
首先我们需要知道证书分为两类`Development`和`Distribution`。顾名思义，一个是开发测试阶段用的，一个是发布的时候用的。

### Distribution Certification
发布证书的工作原理比较简单，因为我们的App都是通过App Store下载的，所有App都需要上传到Apple的服务器。Apple拥有一对公私钥，Apple使用私钥签名每一个上传到App Store的App。每台设备上又拥有一份公钥的备份，在运行App前iOS系统会使用这个公钥去验证App的签名是否合法，从而保证了即将运行的App是受信任的。
<img src="diagram.001.png" alt="Distribution Certification">
1. 开发者提交App到Apple服务器，Apple使用私钥签名App生成安装包。
2. 运行时系统使用公钥验证App的签名是否合法，若不合法，则阻止App启动。

### Development Certification
`Development Certification`的工作原理比起`Distribution Certification`则稍微复杂一些，因为我们的App是直接被安装到设备上而不是通过App Store。创建`Development Certification`的时候，需要先在本地申请一个`Certificate Signing Request`文件，这实际是生成了一对密钥，然后把公钥传给Apple，最后从Apple那下载一份Certification。这一过程的原理和上传App到App Store有些相像，只不过之前我们验证的是App的签名，这里变成了验证开发者的公钥签名。具体的流程如下：
<img src="diagram.002.png" alt="Distribution Certification" width="70%" height="70%">

1. 我们在本地申请`Certificate Signing Request`并把*Pub<sub>m</sub>*上传给Apple。
2. Apple使用*Pri<sub>A</sub>*签名*Pub<sub>m</sub>*，获得一份证书，这份证书包含*Signature(Pub<sub>m</sub>)*和*Pub<sub>m</sub>*。
3. 使用*Pri<sub>m</sub>*签名App获得*Signature(App)*，连同`2`中生成的证书和App一起打包成IPA文件。
4. Device上打开App时，系统使用*Pub<sub>A</sub>*验证证书，验证成功则表明*Pub<sub>m</sub>*是受Apple信任的。
5. 使用受信任的*Pub<sub>m</sub>*去验证*Signature(App)*，确保它是由*Pri<sub>m</sub>*签名的。

这样一来，就间接的验证了App是被Apple所信任的。

## Provisioning Profile
除了证书之外，最后的安装包还会包含`entitlement`，`App ID`，如果是Development的安装包，那么还会包含`Device ID`s。Apple把这些文件的集合称之为`Provisioning Profile`。它最后会以`embeded.mobileprovision`存在于`IPA`文件内。
<img src="image.png" alt="Provisioning Profile">

### Development Provisioning Profile
`Development Provisioning Profile`的工作原理和`Development Certification`有些相像，可以理解成Provisioning Profile只是往里面多添加了一部分数据。
<img src="diagram.003.png" alt="Distribution Certification" width="70%" height="70%">

1. 我们在本地申请`Certificate Signing Request`并把*Pub<sub>m</sub>*上传给Apple。
2. Apple使用*Pri<sub>A</sub>*签名*Pub<sub>m</sub>*，获得一份证书*Cert*，这份证书包含*Signature(Pub<sub>m</sub>)*和*Pub<sub>m</sub>*。
3. Apple使用*Pri<sub>A</sub>*签名*Cert*，*Entitlement*，*App ID*以及*Device ID*s，获得Provisioning Profile*PP*。
4. 使用*Pri<sub>m</sub>*签名App获得*Signature(App)*，连同*PP*和App一起打包成IPA文件。
5. Device上打开App时，系统使用*Pub<sub>A</sub>*验证*PP*，验证成功后再验证*PP*里的证书*Cert*。
6. 使用证书中的*Pub<sub>m</sub>*验证*Signature(App)*。

### Distribution Provisioning Profile
`Distribution Provisioning Profile`有多种类型的`Provisioniong Profile`
- Ad Hoc
- In House
- App Store

`Ad Hoc`和`In House`的流程和Development的原理是差不多的。`App Store`的工作原理比较简单，和上面讲的差不多，因为可以直接使用私钥对App进行签名。[这里](https://wereadteam.github.io/2017/03/13/Signature/)有详细的说明
> 而 AppStore 的签名验证方式有些不一样，前面我们说到最简单的签名方式，苹果在后台直接用私钥签名 App 就可以了，实际上苹果确实是这样做的，如果去下载一个 AppStore 的安装包，会发现它里面是没有 embedded.mobileprovision 文件的，也就是它安装和启动的流程是不依赖这个文件，验证流程也就跟上述几种类型不一样了。

> 据猜测，因为上传到 AppStore 的包苹果会重新对内容加密，原来的本地私钥签名就没有用了，需要重新签名，从 AppStore 下载的包苹果也并不打算控制它的有效期，不需要内置一个 embedded.mobileprovision 去做校验，直接在苹果用后台的私钥重新签名，iOS 安装时用本地公钥验证 App 签名就可以了。

> 那为什么发布 AppStore 的包还是要跟开发版一样搞各种证书和 Provisioning Profile？猜测因为苹果想做统一管理，Provisioning Profile 里包含一些权限控制，AppID 的检验等，苹果不想在上传 AppStore 包时重新用另一种协议做一遍这些验证，就不如统一把这部分放在 Provisioning Profile 里，上传 AppStore 时只要用同样的流程验证这个 Provisioning Profile 是否合法就可以了。

> 所以 App 上传到 AppStore 后，就跟你的 证书 / Provisioning Profile 都没有关系了，无论他们是否过期或被废除，都不会影响 AppStore 上的安装包。


## Reference
[iOS App 签名的原理](https://wereadteam.github.io/2017/03/13/Signature/)
[逆向Appstore应用（二）](https://www.jianshu.com/p/f42073b4bea1)
[iOS - How to Create a Provisioning Profile](https://customersupport.doubledutch.me/hc/en-us/articles/229496268-iOS-How-to-Create-a-Provisioning-Profile)