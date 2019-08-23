---
layout: post
title: 'Raspberry Pi初体验'
categories: Tech
tags:
  - RaspBerry Pi
---

原来是想弄个国内VPS自己跑点服务在上面玩玩。试用了腾讯云的主机，体验还是挺不错的，可是奈何穷，随便一年就得大几百。偶然发现了树莓派这个玩意，觉得可以自个搭个微型的服务器耍耍，没有太多的需求，跑跑小服务，足矣。以后还能捣鼓捣鼓智能家居，岂不是美滋滋。马上淘宝整了一套最基本的套件，今天到了，迫不及待的体验一发。
<!-- more -->

我只购买了一块最新版的`Raspberry Pi 3B+`、一个亚克力的外壳、一块TF卡以及一个电源。

拿到手，首先当然是给我们TF卡刷入系统啦，图个方便省事，刷一个树莓派官方的系统，[这里](https://www.raspberrypi.org/downloads/)就能下到最新系统，按照教程使用`Etcher`写入我们的TF卡。

等待烧录完成，将TF卡插入我们的板子，接通电源，插上网线。去路由器里找到树莓派的IP地址，使用`SSH`登录。这里需要注意，新版的系统默认禁用了`SSH`登录。要开启`SSH`，需要新建一个空白的文件在TF卡的根目录下，并命名为`ssh`。这样我们就能通过`SSH`登录我们的树莓派系统啦。
默认的用户名是`pi`，密码是`RaspBerry`。
```Shell
ssh pi@[ip address]
```

登录完成，第一件事当然是修改掉默认的密码:
```Shell
sudo passwd pi
# enter new password for pi
sudo passwd root
# enter new password for root
```

使用不惯`vi`和`nano`，安装`vim`：
先更新一下
```Shell
sudo apt-get update
```
然后安装`vim`
```Shell
sudo apt-get install vim
```

不想让树莓派一直连着网线，想使用`WiFi`的网络环境。也很简单，找到`/etc/wpa_supplicant/wpa_supplicant.conf`
```Shell
sudo vim /etc/wpa_supplicant/wpa_supplicant.conf
```

在文件的末尾添加`WiFi`信息：
```Shell
  network={
  ssid="SSID_NAME"
  psk="WiFi_PASSWORD"
  priority=5
}
```
`ssid`是你希望连接到的`WiFi`的名字，`psk`是`WiFi`的密码，`priority`是连接的优先级，越大优先级最高。因为这个文件中可以配置多个`network`的信息。

由于我想要在上面跑一些`Python`的脚本，所以需要安装`Python`的环境。新版的系统已经安装了`python`和`python3`，但是没有`pip`。
```Shell
sudo apt-get install python3-pip
```
安装好了`pip`，就可以安装茫茫多的`Python`库啦。