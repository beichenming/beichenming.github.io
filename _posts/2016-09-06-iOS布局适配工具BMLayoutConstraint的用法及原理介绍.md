---
layout: post
title: iOS布局适配工具BMLayoutConstraint的用法及原理介绍
subtitle:   "App 应用笔记"
date:       2016-09-06 21:31:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - Object C
    - iOS
---

`原创文章转载请注明出处，谢谢`

---

记得之前写过一篇iOS中关于不同设备布局适配方案的文章，然后有不少同学问我要Demo，由于前段时间比较忙，趁着G20放假七天于是就整理了一下之前的代码。从之前写这套方案到现在，使用了三个月效果还是很理想的，不过中途还是发现了之前的方案并不能适配同一设备多语言下不同的适配规则，于是我就在原来的基础上进行了改版，增加了同一设备多语言下的适配方案。于是就写了上面所说的BMLayoutConstraint这个工具，下面我介绍一下这个工具的用法以及原理。[BMLayoutConstraint](https://github.com/beichenming/BMLayoutConstraint)

#### 前沿
写BMLayoutConstraint的初衷是因为我们的项目的界面并没有使用xib，storyboard作为方案，而是通过Masonry配合纯代码的方式编写的界面，所以需要不同设备下的适配方案来降低代码的耦合度，至于iOS下关于适配方案的总结我在之前的文章中就有提到，没看过的同学可以先去看看。[iOS自定义不同设备下的UI布局](http://beichenming.me/2016/07/13/iOS%E8%87%AA%E5%AE%9A%E4%B9%89%E4%B8%8D%E5%90%8C%E8%AE%BE%E5%A4%87%E4%B8%8B%E7%9A%84UI%E5%B8%83%E5%B1%80/)

#### 关于原理
BMLayoutConstraint的主要思想就是通过配置文件读取相应的适配代码，从而适应不同屏幕的约束。

BMLayoutConstraint的结构很简单主要是由以下几个文件组成：

* BMLayoutConstraint.h (包含了所有的头文件)
* BMTLayoutConstraintBase.m (定义了用于解析JSON文件的控件基本属性)
* BMTLayoutConstraintConstant.m (关于设备常量大的定义)
* BMTLayoutConstraintInterpreter.m (JSON解释器)
* BMTLayoutConstraintLanguage.m (关于多语言适配的定义)
* UIView+LayoutConstraint.m (UIView的扩展，用于读取控件的属性)

大致的原理我已经在[iOS自定义不同设备下的UI布局](http://beichenming.me/2016/07/13/iOS%E8%87%AA%E5%AE%9A%E4%B9%89%E4%B8%8D%E5%90%8C%E8%AE%BE%E5%A4%87%E4%B8%8B%E7%9A%84UI%E5%B8%83%E5%B1%80/)中介绍过了，所以这里主要讲针对于多语言的适配原理。

我采取的方案是在所有的ID结尾追加一个多语言的标识符，对于同一设备下多语言的适配不变的情况，我们使用默认的BM_BASE；对于同一设备下多语言的适配会改变的情况，我们使用每种语言的标识符作为结尾。

```
 "UILabel" :
    [
         {
             "bm_ViewControllerPhoneNoLabelID_BM_ZH_HANS_US" :
             {
                 "marginLeft" : 100.0,
                 "marginRight" : 0.0,
                 "marginTop" : 300.0,
                 "marginBottom" : 0.0,
                 "width" : 250.0,
                 "height" : 30.0,
                 "fontSize" : 32.0
             }
         },
         {
             "bm_ViewControllerPhoneNoLabelID_BM_EN_US" :
             {
                 "marginLeft" : 50.0,
                 "marginRight" : 0.0,
                 "marginTop" : 300.0,
                 "marginBottom" : 0.0,
                 "width" : 250.0,
                 "height" : 30.0,
                 "fontSize" : 32.0
             }
         },
    ]

```

但是其中有一些小细节需要我们特殊处理：

1. 对于一个控件，如果我们设置了以BM_BASE作为结尾的ID，那么我们就会把该ID作为默认的约束准则，即使设置了其他多语言的约束也不会生效。
2. 如果我们能在系统的语言优先级中找到符合我们App的语言，我们就已该语言作为我们的约束规则。
3. 如果我们能在系统的语言优先级中不能找到符合我们App的语言，我们就通过CFBundleDevelopmentRegion设置的默认语言作为我们的约束条件。

以下的一段代码就是我们对于多语言下优先处理规则

```
  // exist common layout constraint
    NSString *defaultConstraintDeviceLanguage =
    [self.layoutConstraintLanguage getLayoutConstraintDefaultDeviceLanguage];
    NSString *defaultViewConstraintId =
    [NSString stringWithFormat:@"%@_%@", viewConstraintId, defaultConstraintDeviceLanguage];
    if ([self cacheConstraintMapDictObjectForKey:defaultViewConstraintId]) {
        return defaultViewConstraintId;
    }
    
    // find mutal language layout constraint
    NSArray *languageArray = [NSLocale preferredLanguages];
    for (NSString *language in languageArray) {
        NSString *constraintDeviceLanguage =
            [self.layoutConstraintLanguage.preferredLanguagesDict objectForKey:language];
        NSString *newViewConstraintId =
            [NSString stringWithFormat:@"%@_%@", viewConstraintId, constraintDeviceLanguage];
        if ([self cacheConstraintMapDictObjectForKey:newViewConstraintId]) {
            return newViewConstraintId;
        }
    }
    
    // user default Language
    NSString *defaultLanguage =
    [[[NSBundle mainBundle]infoDictionary] objectForKey:@"CFBundleDevelopmentRegion"];
    NSString *constraintDeviceLanguage =
    [self.layoutConstraintLanguage.defaultLanguagesDict objectForKey:defaultLanguage];
    NSString *newViewConstraintId =
    [NSString stringWithFormat:@"%@_%@", viewConstraintId, constraintDeviceLanguage];
    if ([self cacheConstraintMapDictObjectForKey:newViewConstraintId]) {
        return newViewConstraintId;
    } else {
        NSLog(@"%@ not in Localization native development region(CFBundleDevelopmentRegion)", defaultLanguage);
        NSLog(@"use this (Canada, Canada(French), China, France, Germany, Italy, Japan, Korea, Taiwan, United Kingdom, United States)");
        assert(0);
    }

```

#### 示例
[详见](https://github.com/beichenming/BMLayoutConstraint)

#### 总结
如果大家觉得有收获，希望可以帮我点颗star，谢谢




