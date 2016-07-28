---
layout: post
title: MacOSX&iOS操作系统之版本迭代史
subtitle:   "深入解析MacOSX&iOS操作系统笔记"
date:       2016-07-17 12:45:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - 操作系统
---
`原创文章转载请注明出处，谢谢`

---

#### 1.关于Mac OS Classic
Mac OS Classic是Mac OSX在OSX之前时代的名字，Mac OS Classic最后一个版本就是Mac OS 9，这也就是为什么OSX都是从10开始的原因，Mac OS Classic是一个全GUI的操作系统（但是并没有Terminal这样的应用存在）。
优点：出现了Finder的GUI，以及第一代HFS文件系统对于fork的支持。
缺点：Mac OS Classic是协作式多任务管理，当有进程拒绝协作的时候，整个系统就会直接停止运行。

#### 2.关于NeXTSTEP
NeXTSTEP也是一个操作系统，由于它才会有后来的Mac OSX，NeXTSTEP拥有了现在我们回过头来比较熟悉的一些特性。

* NeXTSTEP采用的是Mach微内核系统。
* 开发语言采用的是Object-C（这也就是为什么会有NSFoundation）
* NeXTSTEP提供了很多Framework和Kit，开发者可以使用丰富的对象库进行快速GUI开发，这些对象都是基于NSObject。
* 提供了一个DriverKit的驱动开发框架，采用的面向对象的开发模式，Driver可以是其它Driver的子类，可以继承或扩展其它Driver的功能。
* Application和Famework已bundle的形式发布(是不是很熟悉)。bundle由一个固定的目录结构组成，用于封装一个软件包，其中带有自己所需要的依赖和相关文件，因此软件的安装和删除就像移动一个目录一样简单。
* 系统中重度使用PostScript。

#### 3.关于Mac OSX
OSX是Mac OS Classic和NeXTSTEP相结合的产物，最后NeXTSTEP慢慢吸收了前者，最终形成了Mac OSX。OSX的核心组件-Cocoa,Mach,IOKit,Xcode的Interface Builder以及很多其他组建都来自于NEXTSTEP，Mac OS Classic拥有伟大的GUI但是设计却很糟糕，NEXTSTEP设计精良但是GUI却平淡无奇，最后致使它们结合到了一起，形成了伟大的Mac OS X系统。

`到目前为止，Mac OSX从10.0更新到了10.11。`

**10.0 Cheetah（猎豹）**

开始正式整合NeXTSTEP，支持抢占式多任务管理和内存保护。对Mac OS 9彻头彻尾的重写，它和Mac OS9共有的部分很少，可能只保留Carbon接口，这是为了维持OS9 API的兼容性。

**10.1 Puma（美洲狮）**

提高了稳定性和性能，以及用户体验，也正式放弃了Mac OS 9。

**10.2 Jaguar（黑豹）**

引入了"Quartz Extreme"框架用于图形快速显示，以及Bonjour(Rendezvous)协议，允许Apple的设备可以在同一个局域网里相互发现。

**10.3 Panther（黑豹）**

正式推出了Safari，不在开始使用微软提供的浏览器。
以及增强了FileVault，实现了透明的磁盘加密。

**10.4 Tiger（老虎）**

出现了SpotLight(搜索工具)以及Dashboard两个工具。正式使用Intel x86处理器，放弃使用PowerPC的架构（并没有完全抛弃）。同时引入了"通用二进制"的概念，通用二进制的代码既可以在PowerPC上运行，也可以在Internal x86上运行，同时支持64位指针。同时添加了CoreData，CoreAudio，CoreImage，CoreVideo几个重要的框架。

**10.5 Leopard（花豹）**

Leopard更新了大量的新特性：
添加了CoreAnimation，用来处理界面动画。推出了Object-C 2.0，以及OpenGL 2.1，增强了脚本语言，还增加了Python和Ruby的语法支持。同时开始完全遵循UNIX/POSIX规范。DTrace（从Solaris 10移植而来）及对应的GUI和Instrument。FSEvents，实现了类似Linux的inotify的功能（文件系统和目录相关通知）。

**10.6 Snow Leopard（雪豹）**

完整的64位支持，所有的自带应用程序都移植到了64位，同时包括所有的用户态的库以及内核空间。以及文件系统层次的压缩，节省了用户的磁盘存储。GCD实现了对多核编程的支持。同时将OpenCL的计算负载转移到了GPU上。这个版本开始正式放弃使用PowerPC的架构，完全使用x86/64架构，同时也不需要在使用通用二进制。

**10.7 Lion（狮子）**

推出了iCloud服务。开始使用了Sandbox（来源于iOS）和权限分离模型。推出了FaceTime和LaunchPad（来源于iOS）。使用了CoreStorage，支持逻辑卷，可以用于分区特性，支持将文件系统扩展到多个分区上。推出了AirDrop，允许主机间通过Wifi快速分享文件（基于Bonjour协议）。在更多机器上默认开启64位模式，因为还有存在部分机器采用了32位的内核。

**10.8 Mountain Lion（美洲狮）**

内核开始只支持64位，抛弃了对32位API的支持。

#### 4.关于iOS

**iOS1.0**

第一代的iPhone拥有完整的调试符号，同时没有加密，还容易反汇编。(奠定了以后越狱的基础)

**iOS2.0**

正式推出了AppStore，支持VPN和Microsoft ExChange，同时支持大量其他语言。

**iOS3.0**

支持第一代iPad设备，同时支持个人热点。支持剪切/粘贴功能，支持小众的语言，支持Spotlight搜索。

**iOS4.0**

支持iPhone4，Apple TV 和iPad2。支持FaceTime和语音控制，支持真正的多任务版本。

**iOS5.0**

推出了iPhone 4S。推出Siri，支持通知功能，iCloud。

**iOS6.0**

1. 抛弃Google Map，使用自家的地图，MapKit也开始和Apple的地图开始绑定，第三方的App可以和地图进行交互。
2. 深度社交网络的集成，包括Facebook和Sina Weibo，直接可以通过调用Social.framework来完成，同时新增了一个UIActivityController来询问用户的社交行为。
3. Passbook可以用来存储一些优惠券电影票等，可以使用NFC完成电子钱包等功能；PassKit可以用来生成和读取包含一些类似优惠券之类信息的特殊文件，然后以加密签名的方式发送给用户。
4. Game Center的一次升级
5. 自带的提醒应用得到加强，开放了Reminder里添加东西和从中读取的API，以及一套标准的用户界面。
6. 提供了新的瀑布流展示方式，UICollectionViewController。
7. UI状态的保存；Apple希望用户关闭app，然后下一次打开时能保持关闭时的界面状态。对于支持后台且不能被kill的情况是天然的，但是如果不支持后台运行或者用户自己kill掉进程的话，就比较麻烦。现在的做法就是从rootViewController开始把所有的VC归档后存成NSData，然后下次启动的时候做检查如果需要的话就恢复出来。
8. 整个UIView都支持NSAttributedString格式化字符串。特别是UITextView和UITextField。

**iOS7.0**

1. 全新的UI设计；状态栏，导航栏和应用实际展示内容不再界限：系统自带的应用都不再区分状态栏和navigation bar，而是用统一的颜色力求简洁；BarItem的按钮全部文字化；程序打开加入了动画：从主界面到图标所在位置的一个放大，同时显示应用的载入界面；
2. UIKit方面的力学模型，以及游戏方面加入了Sprite Kit Framework和Game Controller Framework；
3. 可以通过设置UIBackgroundModes为fetch来实现后台下载内容了，需要在AppDelegate里实现setMinimumBackgroundFetchInterval:以及application:performFetchWithCompletionHandler:来处理完成的下载。
4. AirDrop；用户可以用它来分享照片，文档，链接，或者其他数据给附近的设备。但是不需要特别的实现，被集成在了标准的UIActivityViewController里。
5. 增加了自带地图的一些特性；
6. AudioUnit框架中加入了在同一台设备不同应用之间发送MIDI指令和传送音频的能力；
7. 新增Touch ID；

**iOS8.0**

1. 应用扩展（Extension），Apple 允许我们在 app 中添加一个新的 target，用来提供一些扩展功能：对于应用扩展，Apple 将其定义为 App 的功能的自然延伸，因此不是单独存在的，而是随着应用本体的包作为附属而被一同下载和安装到用户的设备中的，用户需要在之后选择将其开启。另外，由于应用扩展和应用是属于两个不同的 target 的，因此它们之间的数据和操作上的交互遵循的是另一套原则。
2. 添加了sizeclass的布局方式，可以使用一套UI来适配不同的屏幕。
3. 新增了HealthKit和HomeKit的应用，以及框架。
4. Touch ID API开放
5. iMessage可以发送语音，视频；开发Touch ID功能，
6. 改进了Spotlight的本地搜索功能。

**iOS9.0**

1. Multitasking多任务特性，特别是分屏多任务使得 iPad 真正变得像一个堪当重任的个人电脑。
2. 在新的 watchOS 2 中，Watch App 的架构发生了巨大改变。新系统中 Watch App 的 extension 将不像现在这样存在于 iPhone 中，而是会直接安装到手表里去，Apple Watch 从一个单纯的界面显示器进化为了可执行开发者代码的设备。
3. 引入了UI Test
4. Swift2.0
5. HealthKit引入新的体征，以及HomeKit和CloudKit也有不少变化。

#### 总结一下Mac OS X和iOS的一些差异
`1.iOS内核和二进制文件编译的目标架构是基于ARM架构的，OSX是基于Intel i386和x86_64的（现在应该都是x86_64），iOS的处理器目前最新的是A9处理器，不管是哪一代都是采用ARM设计的，ARM相比Intel的优势在于电源管理方面。`

`2.iOS的GUI是SpringBoard，属于触屏应用加载器，而Mac OSX中的GUI采用的是Aqua，属于鼠标驱动，是为窗口程序设计的。在OSX10.7中SpringBoard以Launchpad的形式移植到了OSX。`

`3.iOS对于系统管理的权限相比OSX更加的严格，应用程序不允许访问底层的UNIX_API(Darwin)，同时也没有Root访问权限，只能访问自己目录里的数据，只有Apple的原生应用才有权限访问全系统目录的权限。`
`当然其他还有很多其他特性...`