---
layout: post
title: 关于如何用iOS逆向解决现实中的问题
subtitle:   "逆向"
date:       2020-04-12 07:31:00
author:     "BCM"
header-img: "../../img/post-bg-unix-linux.jpg"
tags:
    - 逆向
---

`原创文章转载请注明出处，谢谢`

---


## 前沿

近期遇到了一个AppStore上App升级的问题，最后捣鼓了两天时间才解决问题，其中用到不少逆向的知识，所以这篇文章也算是简单聊一下逆向相关的知识点吧；由于牵扯到的有点多，很多操作细节我就不详述了大家可以自行google;

## 获取App

最近我们“有计划有预谋”的想要在**AppStore**上更新一个三年以前的老包，但是由于历史原因，再加上多人易手，当初的代码已经找不回来，唯一知道的也就只有开发者账号了；

所以第一步就是需要在**Appstore**上把目前最新的老包下载下来，然后分析一下，主要也就是看看之前的账号体系能不能在新版本被保留下来；

说到从**AppStore**下载**ipa**包，在很早的那个年代也就是**iTunes 12.7**版本之前，当时由于**iTunes**自带了**AppStore**，所以可以很轻松的就下载到需要的**ipa**包；早期的时候还可以通过安装**iTunes 12.7**以前的旧包来进行操作，不过后面由于**OSX系统**的不断升级，早期的**iTunes**包就无法安装了；所以目前来说，唯一可行的办法就是通过**Apple Configurator 2**了；

<p align="center">
<img src="../../../../img/technology/2020-04-12/apple_configurator_2.png" alt="change_pic" title="change_pic"/>
</p>

通过**Apple Configurator 2**得到包以后，这个包是不能够直接安装到手机上面的，**AppStore**下载的**App**全都是经过苹果加密过的**ipa**包，由于**AppStore**使用**Apple ID**相关的对称加密算法，因此加密后的**ipa**包是无法对其进行反编译的，也无法**class-dump**，需要对其进行解密才能反编译，也就是常说的砸壳；

不过砸壳的前提就是需要一台越狱机；

## 关于越狱

关于**越狱**，前段时间**PP助手**正式下线了，也代表着**iOS越狱市场**结束了；毕竟随着**iOS系统**版本的不断迭代，以及**Appstore**上产品的不断丰富，之前只能通过**越狱**才能有的**App**现在直接就能下载到了，侧面证明了有需求才有市场的道理；不过对于我们这种开发人员而言，越狱手机确实能够帮我们解决不少的问题；

<p align="center">
<img src="../../../../img/technology/2020-04-12/pp_helper.png" alt="change_pic" title="change_pic"/>
</p>

 **越狱市场**的萎缩也和**iOS系统**越来越难破解有着密不可分的关系，目前来说**iOS8**以及**iOS9**两个系统应该是**越狱**最稳定的两个系统了；**iOS10**系统好像区分**完美越狱版**和**非完美版**，10以上的系统由于没有试验过，所以这里不下结论，市面比较常见的就是**盘古**和**太极**两个越狱工具了，这方面其实对于各个手机型号以及具体的系统子版本都是有区分的；
 
 由于测试设备的限制，我手头上也就只有一个**iOS10.2**的**iPhone6**手机能够**越狱**了，考虑到我并不想做到完全把手机越狱的程度，选择了**非完美版**；这里说的**非完美版**的意思就是每次手机重启以后就会恢复到**非越狱**的状态，然后需要重新再跑一遍**越狱脚本**；
 
 以**iOS10.2**为例，一般常用的越狱软件就是**yalu102**；
 
 <p align="center">
<img src="../../../../img/technology/2020-04-12/yalu102_go.png" alt="change_pic" title="change_pic"/>
</p>
 
 
 用过的人应该知道首先你需要用**Cydia Impactor**对**yalu102**进行自签名，不然是无法安装的，一般需要用自己的**AppleID**对其进行签名；之后就可以用**itools**之类的工具对**ipa**包进行安装了，安装完成以后就可以用**yalu102**进行越狱了，越狱成功以后我们常见的**Cydia**也就安装完成了；
 
  <p align="center">
<img src="../../../../img/technology/2020-04-12/cydia_impactor.png" alt="change_pic" title="change_pic"/>
</p>


## 关于砸壳

当**越狱**以后就可以进行**砸壳**的操作了，常见的砸壳工具一般是**Clutch**和**dumpdecrypted**这两种，这两个工具在**Github**上都能够找到；

  <p align="center">
<img src="../../../../img/technology/2020-04-12/clutch.png" alt="change_pic" title="change_pic"/>
</p>

  <p align="center">
<img src="../../../../img/technology/2020-04-12/dumpdecrypted.png" alt="change_pic" title="change_pic"/>
</p>

砸壳结束以后我们就能够得到一个完整的没有加密过的**ipa**包了；对于一般的逆向工程来说，接下去常见的就是去反编译里面的二进制文件，会使用到也就是比如**class-dump**，**Hopper disassembler v4**等这些工具；

  <p align="center">
<img src="../../../../img/technology/2020-04-12/class_dump.png" alt="change_pic" title="change_pic"/>
</p>

  <p align="center">
<img src="../../../../img/technology/2020-04-12/hopper_v4.png" alt="change_pic" title="change_pic"/>
</p>

## 关于重签

**重签**的问题其实上面有提到过，在使用**yalu102**的时候就需要用**Cydia Impactor**对其进行**重签**；不过我个人不推荐用这种方式，因为如果是使用那个**非完美**方式的**越狱**，每次重启手机以后又回恢复到之前的**非越狱**状态，这个时候由于签名的问题**yalu102**也就打不开了，需要重新连接电脑在进行一次**越狱**；

所以我推荐可以使用自己的开发证书对**yalu102**进行**重签**，这样就不会出现过期打不开的问题；**重签**的方式一般就是**Xcode**自带的**codesign**命令，如果觉得使用不方便的话，我推荐使用一个工具，就是**iOS App Signer**；这个工具对于签名相当好用，不过我建议大家在使用的时候最好还是断网，虽然我监控过它并没有把我们的证书往服务端传，不过还是小心为好，顾虑安全的同学还是不要使用；原因在于普通的开发证书还行，要是**企业证书**被人拿走了那损失就难以估计了，毕竟**企业证书**已经炒到天价了，而且现在用一个少一个；

  <p align="center">
<img src="../../../../img/technology/2020-04-12/ios_app_sign.png" alt="change_pic" title="change_pic"/>
</p>


## 关于Appstore旧包

我们把新包更新上去发现老包更新以后账号还是丢失了；由于之前我是将老包重签以后覆盖更新的测试账号升级的，发现这样并没有什么问题；所以怀疑还是其他地方出了问题，由于我们的测试设备都已经升到最新版的App，而之前**重签**的旧包又不能复现这个问题，所以只能想办法让**越狱机**能够下载到上个版本的旧包才行；

这个时候需要的工具就是用**Cydia**下载一个叫做**App admin**的插件，俗称**App降级插件**；需要注意的是**App admin**也是有版本支持的，需要根据自己不同的系统版本进行对应的下载，由于我自己的是**iOS10**，之前下载的版本只支持到**iOS9**，于是就会**Crash**；

下载完**App admin**以后会对**Springboard**进行重启，然后在打开**AppStore**时候，在**App**对应的下载界面长按下载就会出现**降级**的按钮，然后用户就可以根据需要下载不同的版本了；值得一提的是，你需要先安装对应的**App**才能进行降级，不能直接下载旧版本；下载完成以后我们就可以用**iFunbox**来查看对应的目录结构;

  <p align="center">
<img src="../../../../img/technology/2020-04-12/app_admin.png" alt="change_pic" title="change_pic"/>
</p>


## 关于Keychain的查看
最后发现可能是**Keychain**存储上出现的问题，于是就需要把**Keychain**从手机中导出来；一般我们使用的工具就是**Keychain-Dumper**；系统保存**keychain**的目录一般都是在**/private/var/Keychains**下面，把**Keychain-Dumper**放在**bin**目录下，然后在**keychain**目录执行就可以了；

  <p align="center">
<img src="../../../../img/technology/2020-04-12/keychain_dumper.png" alt="change_pic" title="change_pic"/>
</p>

   <p align="center">
<img src="../../../../img/technology/2020-04-12/key_chain.png" alt="change_pic" title="change_pic"/>
</p>

## 总结
以上就是差多解决问题用的到一些基础逆向知识，其实平时开发中用到逆向的场景挺多的，比如马甲包等等，之后还会介绍这一块的内容；
 
 







