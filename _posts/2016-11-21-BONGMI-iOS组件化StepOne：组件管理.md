---
layout: post
title: BONGMI-iOS组件化StepOne：组件管理
subtitle:   "App 应用笔记"
date:       2016-11-21 21:31:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - Object C
    - iOS
---

#### 前沿
许久没有更新博客了，十一回来以后公司研发部门就着手开始了组件化的工作，其中涉及到App以及服务器，可以说是一次大规模的改动；中间也是踩了不少的坑，到目前为止App iOS这边也只是完成了组件化第一部分的内容。虽然网上有很多讲关于组件划分，以及组件解耦这些方面的知识，但是我基本没见过讲组件管理方面的一些事，当组件划分结束后第一件会碰到的就是关于组件管理，所以我觉得有必要需要梳理一下之前所遇到的一些问题，希望可以帮到一些同学。

#### 历史背景
首先我需要介绍一下我们组件化的一些历史背景，一般来说当App需要组件化的时候主要原因有两方面：

* 降低代码耦合，提升开发效率。随着工程代码的日益增加，以及开发人员的增加，如果只是单一的维护一个项目工程会非常影响开发效率，增加开发成本。
* 提取公共模块，复用代码。如果公司存在多个App，那么这些App之前一定存在着一些公共的模块，所以我们需要把这些相同的代码提取出来做成一些模块，那么每个App只要包含这些公共的模块就可以了。

然后就我们目前自身情况的分析，我们组件化的目的在于提取公共模块复用代码，因为我们在接下去会开发一款新的App，里面包含了不少和目前App相同的部分，所以就需要将这部分提取出来。

#### 组件管理
对于组件的划分，每个项目都有自己的特点所以这方面我就不多讲了，以下图为例，这么我们App目前划分的其中一部分设计，主要目的是为了方便我后面的一些内容讲解。

在iOS中最常用到Cocoapod用来做组件化的管理（当然现在好像还有一个叫做Carthage的工具也可以），个人认为使用Cocoapod用作组件管理的好处有以下几个方面：
	
	* 简单的组件版本控制
	* 简单的静态库打包工具
	* 简单的第三方库管理调用

---

好，接下来我们就具体来讲组件管理方面的一些事！

记得很早以前的维护的一个Mac项目（最早的代码年纪比我还大），上面的组件化基本都是以target的方式进行，Project的编译促使所有的target进行编译，这种方式的组件化缺点在于虽然从代码程度上实现了解耦，但是每个组件依然还是在主工程下，同时单独的每个组件不方便做单元测试，所以到现在这种方式已经不适合做组件化的管理了。

我们的理想效果应该是每个组件有单独的工程，同时包含每个模块的单元测试，并且每个组件方便进行不同的版本替换。所以综上所述使用Cocoapod就非常适合了。

Cocoapod对于库的创建一般有两种形式pod spec和pod lib这两种，前者用于创建一个单独的podspec文件，后者用于静态库的创建同时会生成相应的模版。但是最后只要存在podspec文件，那就都可以打包成静态库，本质上说两者没有太大的区别。

这里有一个概念上的问题，本来我的计划是希望所有的组件都可以独立编译成一个静态库（这里指的是打包成静态库同时可以使用，不是单纯的通过编译），但是事实上有些组件是不可以的。

举个例子

<p align="center">
<img src="../../../../img/technology/2016-11-21/pic_1.jpeg" alt="change_pic" title="change_pic" width="500"/>
</p>

上图中有一个叫做LollypopBaseNetwork的基础组件，它封装了我们的底层网络通信，上层所有的网络请求都需要依赖这一层，但是它实际的本质也还是依赖的第三方的YTKNetwork，所以存在一个这样的问题：

```
#import "BMBNCancelable.h"
#import "YTKRequest.h"

@class BMBNRequestBase;

typedef void (^BMBNRequestHandler)(BMBNRequestBase *request,
                                   id result,
                                   NSError *error);


@interface BMBNRequestBase : YTKRequest<BMBNCancelable>

@property (nonatomic, copy, readwrite) BMBNRequestHandler bmtHandler;

- (instancetype)initWithMethod:(YTKRequestMethod)method
                          path:(NSString *)path
                     arguments:(id)arguments
                serializerType:(YTKRequestSerializerType)serializerType;

- (void)startWithHandler:(BMBNRequestHandler)handler;

@end
```

我们有一个基础的request的接口，上层中所有的网络请求都需要继承自这个request，但是我们在这个class中#import "YTKRequest.h"，所以当我们把这个组件打包成静态库的时候（即framework或者.a），实际import LollypopBaseNetwork的时候会找不到#import "YTKRequest.h"，因为我们没有把这个第三方的头文件添加到public headers中，原因是我们也不能添加这些第三方的头文件到静态库中防止冲突。所以对于这类组件我们只能通过podfile添加依赖的方式引入工程，而不能直接add这个静态库来完成。

其实通过podfile其实是一种优秀的方式，相比直接依赖静态库而言。

---

然而对于直接依赖工程还是通过podfile来进行依赖是一件值得思考的事情，下面就阐述一下我的一些理解。

如果我们选择工程依赖的话，比较好的一种方式就是通过workspace，把每个组件工程添加到worksapce中，最后通过App工程的编译把它们打包成静态库。

```
workspace 'TestApp.xcworkspace'
project 'TestApp/TestApp.xcodeproj'

target 'TestApp' do
  platform :ios, '8.0'
  project 'TestApp/TestApp.xcodeproj'
  pod 'AFNetworking', "~> 3.0"
end

target 'BMBodyStatusBusinessLib' do
  platform :ios, '8.0'
  project 'BMBodyStatusBusinessLib/BMBodyStatusBusinessLib.xcodeproj'
end

target 'BMDeviceBusinessLib' do
  platform :ios, '8.0'
  project 'BMDeviceBusinessLib/BMDeviceBusinessLib.xcodeproj'
end

target 'BMTemperatureBusinessLib' do
  platform :ios, '8.0'
  project 'BMTemperatureBusinessLib/BMTemperatureBusinessLib.xcodeproj'
end

target 'BMUserBusinessLib' do
  platform :ios, '8.0'
  project 'BMUserBusinessLib/BMUserBusinessLib.xcodeproj'
end

```
这种方式存在的问题就在于第一上下层静态库之间也需要互相依赖，所以这里编译存在一些麻烦的处理；第二就是关于静态库合并的问题，真机和模拟器的静态需要lipo合并才能使用，对于mac应用还要考虑32位和64位静态库的合并；所以我觉得对于App层面上不要将这些组件做成真正意义上的的静态库，它们只需要互相独立，每个组件有自己的版本号控制就可以了。

通过podfile的方式因为不需要将它们做成具体的静态库，所以就可以避免上述的一些问题。

---
对于我们私有的这些组件pod而言，我们需要单独建立一个Cocoapod仓库进行存储，不要把这些组件放到公有的仓库上去，这样其他人就可以看到你的源码了；
这里存在一个概念，就是关于Cocoapod中podfile和podspec这两个文件的关系以及区别：

podfile文件想必大家都应该很熟悉才是，它可以方便管理第三方库的使用，同时它是支持本地pod的引用，通过:path就可以方便的配置，不需要讲pod上传到仓库再通过版本号的方式进行引用。

```
target 'BMBaseNetworkLib’ do
  pod 'JSONModel', '1.0.2'
  pod 'YTKNetwork', '1.1.0'
  pod 'BMBaseWidgetLib', :path => ‘~/BongMi/lollypop-client-ios-base-tools'
end
```

podspec文件其实就是一个pod的配置文件，里面记录这个pod的配置信息，编译依赖，仓库地址以及版本信息等等，本地的.cocoapod文件夹其实就是存储了每个pod的podspec文件(不过是JSON格式)，每个的pod install就是去通过找podspec的仓库地址，再去下载相应的源码。但是这里存在的一个问题是，podspec的并不支持本地pod的引用，因为s.dependency只支持具体版本的依赖。

```
  s.dependency 'BMBaseNetworkLib', '~> 0.1.8'
  s.dependency 'BMBaseStorageLib', '~> 0.1.3'
  s.dependency 'BMBaseWidgetLib', '~> 0.1.2' 
```

所以当我们使用pod package打包静态库的时候还是需要依赖指定的组件版本，并不能直接依赖本地版本进行编译，这是一点比较麻烦的点，但是对于版本控制倒是十分方便。pod package的默认仓库查找是去公共的仓库，所以我们需要带上--spec-sources参数指明使用的仓库。

使用pod package的好处在于它会自动帮我们用lipo合并真机和模拟器生成的太静态库，就不要我们自己手动再去合并。

---

还有一点就是关于每个组件都在单独仓库，如何在主工程中管理依赖的问题？

这个其实是一个相对比较简单的问题了，我们只需要通过git中submodule的概念就可以完成我们的需求，具体我就不细说了。



