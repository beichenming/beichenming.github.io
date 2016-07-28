---
layout: post
title: MacOSX&iOS操作系统之OSX和iOS架构
subtitle:   "深入解析MacOSX&iOS操作系统笔记"
date:       2016-07-27 12:45:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - 操作系统
---
`原创文章转载请注明出处，谢谢`

---
介绍的知识点会有点零乱，因为也是书上看来的，这些知识只能通过不断的加深印象才可以记忆下来！

iOS实际上是完整OSX精简之后的版本，和OSX的主要区别在于iOS的架构是基于ARM，OSX是基于x86_64或者Intel x86

#### OSX和iOS架构概述

基本结构可以分为一下几层:

* 用户体验层:OSX包括Auqa(OSX的GUI是采用鼠标驱动)，Dashboard，Spotlight等；iOS包括SpringBoard(iOS的GUI采用触屏加载驱动)，Spotlight；
* 应用框架层:OSX包括Cocoa，Carbon和Java；iOS包括只有包涵Cocoa Touch，也就是说iOS不支持Carbon和Java；
* 核心框架:主要包括QuickTime，OpenGL等；
* Darwin:操作系统的核心包括内核XNU和UNIX Shell环境；

在上面这些Darwin是完全开源的，是整个系统的基础，同时提供了底层的API；但是上面几层是闭源的；

![1.pic.jpg](../../../../img/technology/2016-07-27/pic_one.jpg)


Darwin的基础架构如下图所示，Darwin实际上是两种技术混合在一起的:Mach和BSD，还包括一些其他的组件，主要是IOKit；

![2.pic.jpg](../../../../img/technology/2016-07-27/pic_two.jpg)

---

#### 用户体验层

下面主要介绍一下几个基础的组件原理

**Auqa**

`Aqua是OSX的GUI，采用鼠标驱动，系统的第一个用户态进程launchd负责启动GUI，支持GUI工作的主进程是WindowServer，有趣的是这个程序是在CoreGraphics.framework中，不过CoreGraphics.framework又是在ApplicationSerivces.framework中；WindowServer的代码实际上不完成任何实际的工作，所有的工作实际上都是CoreGraphics框架中的CGXServer函数完成的；loginWindow进程同样是由launchd启动；`

![3.pic.jpg](../../../../img/technology/2016-07-27/pic_three.jpg)

![4.pic.jpg](../../../../img/technology/2016-07-27/pic_four.jpg)

**QuickLook**

`QuickLook是OSX10.5引入的新特性，允许在Finder中快速预览多种不同类型文件。QuickTime采用的是可扩展的架构，使得大部分工作都可以由插件完成，这些插件都可以用Xcode来创建，后缀为.qlgenerator的bundle；只要将插件放入QuickLook的目录，就可以完成插件的安装。quicklooked是系统的"quicklLook服务器"通
过/System/Library/LaunchAgents/com.apple.quicklook.plist文件在登录的时候启动；`

![5.pic.jpg](../../../../img/technology/2016-07-27/pic_five.jpg)

![7.pic.jpg](../../../../img/technology/2016-07-27/pic_seven.jpg)

**Spotlight**

`Spotlight是在OSX10.4时候引入的一项快速搜索技术，当然这项技术同时也包含在iOS中，Spotlight背后的核心就是一个索引服务器mds，mds在MetaData框架中，而这个框架是系统核心服务器的一部
分/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/Metadata.framework/Versions/A/Metadata ；mds是一个没有GUI的后台服务程序。每当有任何文件操作(创建，修改和删除)发生时，内核都会通知这个后台服务程序。当mds收到通知时，mds会通过工作进程mdworker将各种元数据信息导入数据库，mdworker进程可以加载一个具体的Spotlight Importer(Spotlight导入器)从文件中提取元数据信息。导入器同样可以通过Xcode进行开发，后缀为.mdimporter;`

![6.pic.jpg](../../../../img/technology/2016-07-27/pic_six.jpg)

![8.pic.jpg](../../../../img/technology/2016-07-27/pic_eight.jpg)

![9.pic.jpg](../../../../img/technology/2016-07-27/pic_nine.jpg)

---

#### Darwin简介

`这里只做一些基础的介绍，OSX包含了好几种shell，在OSX和iOS上SSH都是默认禁止的，但是在OSX上可以通过修改配置开启SSH，但是iOS上只有通过越狱才可以开启SSH；iOS和OSX的文件系统也存在少许的差异，OSX和iOS采用的都是HFS+的文件系统，HFS+总是会保存大小写的区别，但是可以对大小写敏感或者不敏感，iOS上就是采用了HFSX，默认是对大小写敏感的，OSX则不是；同时HFS+还可以选择打开日志功能，通过使用日志，文件系统可以在强行卸载，断电的情况下更加健壮，因为日志文件系统通过一个日志记录文件系统事物完成的过程。如果文件系统在挂载时发现日志中包含了事物，那么既可以完成事物，也可以放弃事物。`

---

#### OSX的目录结构

OSX是一个符合UNIX标准的系统，所以一些基本的目录结构是相通的。

* /bin: UNIX中的二进制程序，比如一些常用的UNIX命令
* /sbin: 系统程序，这些二进制文件用于系统管理
* /usr: 目录中包含的bin,sbin和lib。/usr/lib用于存放共享的目标文件(类似于windows中存放dll文件的/windows/system32)
* /etc: 包含了系统的大部分配置文件
* /dev: BSD设备文件
* /tmp: 临时目录
* /var: 各种杂项文件，包括日志文件等等

下面是OSX中特有文件目录

* /Applications: 系统应用程序的默认目录
* /Developer: 如果安装了Xcode，那么所有开发者工具的默认安装位置，但是xcode都是通过App Store下载安装的，所以开发工具都位于Xcode自己的bundle里了,以前还是有这个目录的。
* /Library: 系统应用的数据文件存放在这里
* /System: 系统文件目录，其中包含一个Library子目录，这个子目录基本包含了系统中所有重要的组件
* /Uers: 所有用户在主目录所在的目录，每个用户会在这里有一个单独的目录
* /Volumes: 移动设备以及网络文件挂在的目录
* /cores: 如果启用了核心存储，那么这里就是用来保存核心转储文件，其实这个我也不懂

iOS中一些目录特性

* iOS中没有Users目录，只有一个User目录，单用户的原因
* iOS中没有Volumes目录，因为不需要挂载

---

#### 应用程序和App的一些目录结构知识

介绍目录结构的知识前，必须要提一下关于bundle的概念

`bundle是一种标准化的层次结构，保存了可执行代码以及代码所需的资源，通俗来说bundle可以是一种目录结构就像应用程序的结构一样，也可以是像framework的结构一样是一种文件的目标格式。`

`以下就是应用程序的目录结构`

![10.pic.jpg](../../../../img/technology/2016-07-27/pic_ten.jpg)

`以下App的目录结构，但实际上App的目录结构是很混乱的，app的所有文件都可以丢在根目录中。`

![11.pic.jpg](../../../../img/technology/2016-07-27/pic_eleven.jpg)

---

#### 总结

这些知识都是从书上看来的，很零碎，很难讲，我都是当成一份读书笔记来记的，不过后面的可能开始讲些细节的东西的时候会好一点，知识点不至于那么分散。