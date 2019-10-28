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
每个`App ID`有着对应的`Capability List`，它决定了这个`App ID`对应的App（们）能够开启哪些功能（比如我们常见的地理位置服务，内购，推送...都属于这个范畴）。如果我们的一系列App都拥有相同的`Capability`，使用`Wildcard App ID`就可以避免创建多个`Explicit App ID`。

## Certificate
证书，这大概是最让开发者头疼的东西之一。首先要明白证书是给代码签名用的，通俗一点，只要签了名，就表示这个App是由谁谁谁编写的，出了事是要找上门的。
首先我们需要知道证书分为两类`Debug`和`Distribution`。顾名思义，一个是开发测试阶段用的，一个是发布的时候用的。
不管什么证书，我们在申请的时候都需要申请一个`Certificate Signing Request`文件，其实它就是生成了一对密钥，私钥存在了你的Mac系统中，把公钥传到`Developer Center`上。所以如果想和他人共享证书，那么只需要导出你的私钥，给到他们，然后再倒入到他们的Mac系统中就行了。

## Provisioning Profile
那么我们最后的安装包（`IPA`文件）又是怎么和`Identifier`，`Certification`联系起来的呢？这就需要`Provisioning Profile`了。
`Provisioning Profile`可以看做是`Certificate`和`App ID`的集合。`Provisioning Profile`最后会被打包到我们的`IPA`文件里去，这样系统就能知道当前的App是由谁签名的，它可信不可信，它的`App ID`是什么，它拥有哪些`Capabilities`，能够使用哪些系统服务。
`Provisioning Profile`也分为`Development`和`Production`两种类型。`Development`类型的`Provisioning Profile`还包含了`Devices`的信息，它决定了Debug版本的`IPA`能被安装到哪些`Devices`上。
<img src="image.png" alt="Provisioning Profile">