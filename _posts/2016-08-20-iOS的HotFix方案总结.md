---
layout: post
title: iOS中的HotFix方案总结
subtitle:   "App 应用笔记"
date:       2016-08-20 21:45:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - iOS
---
`原创文章转载请注明出处，谢谢`

---

相信HotFix大家应该都很熟悉了，今天主要对于最近调研的一些方案做一些总结。iOS中的HotFix方案大致可以分为四种：

* WaxPatch(Alibaba)
* Dynamic Framework(Apple)
* React Native(Facebook)
* JSPatch(Tencent)

### WaxPatch
WaxPatch是一个通过Lua语言编写的iOS框架，不仅允许用户使用 Lua 调用 iOS SDK和应用程序内部的 API, 而且使用了 OC runtime 特性调用替换应用程序内部由 OC 编写的类方法，从而达到HotFix的目的。

WaxPatch的优点在于它支持iOS6.0，同时性能上比较的优秀，但是缺点也是非常的明显，不符合Apple3.2.2的审核规则即不可动态下发可执行代码，但通过苹果JavaScriptCore.framework或WebKit执行的代码除外；同时Wax已经长期没有人维护了，导致很多OC方法不能用Lua实现，比如Wax不支持block；最后就是必须要内嵌一个Lua脚本的执行引擎才能运行Lua脚本；Wax并不支持arm64框架。

### Dynamic Framework
动态的Framework，其实就是动态库；首先我介绍一下关于动态库和静态库的一些特性以及区别。

`不管是静态库还是动态库，本质上都是一种可执行的二进制格式，可以被载入内存中执行。`
`iOS上的静态库可以分为.a文件和.framework，动态库可以分为.dylib(xcode7以后变成了.tdb)和.framework。`

`静态库： 链接时完整地拷贝至可执行文件中，被多次使用就有多份冗余拷贝。`

`动态库： 链接时不复制，程序运行时由系统动态加载到内存，供程序调用，系统只加载一次，多个程序共用，节省内存。`

`静态库和动态库是相对编译期和运行期的：静态库在程序编译时会被链接到目标代码中，程序运行时将不再需要改静态库；而动态库在程序编译时并不会被链接到目标代码中，只是在程序运行时才被载入，因为在程序运行期间还需要动态库的存在。`

`总结：同一个静态库在不同程序中使用时，每一个程序中都得导入一次，打包时也被打包进去，形成一个程序。而动态库在不同程序中，打包时并没有被打包进去，只在程序运行使用时，才链接载入（如系统的框架如UIKit、Foundation等），所以程序体积会小很多。`

好，所以Dynamic Framework其实就是我们可以通过更新App所依赖的Framework方式，来实现对于Bug的HotFix，但是这个方案的缺点也是显而易见的它不符合Apple3.2.2的审核规则，使用了这种方式是上不了Apple Store的，它只能适用于一些越狱市场或者公司内部的一些项目使用，同时这种方案其实并不适用于BugFix，更适合App线上的大更新。所以其实我们项目中的引入的那些第三方的Framework都是静态库，我们可以通过file这个命令来查看我们的framework到底是属于static还是dynamic。

### React Native
React Native支持用JavaScript进行开发，所以可以通过更改JS文件实现App的HotFix，但是这种方案的明显的缺点在于它只适合用于使用了React Native这种方案的应用。

### JSPatch
JSPatch是只需要在项目中引入极小的JSPatch引擎，就可以使用JavaScript语言调用Objective-C的原生接口，获得脚本语言的能力：动态更新iOS APP，替换项目原生代码、快速修复bug。但是JSPatch也有它的自己的缺点，主要在由于它要依赖javascriptcore,framework,而这个framework是在iOS7.0以后才引入进来，所以JSPatch是不支持iOS6.0的，同时由于使用的是JS的脚本技术，所以在内存以及性能上面是要低于Wax的。

**所以最后当然还是采用了JSPatch这种方案，但是实际过程中还是出现了一些问题的，所以掌握JSPatch的核心原理对于我们解决问题是非常有帮助的。**

### 关于JSPatch的核心原理讲解

---

#### 预加载部分
关于核心原理的讲解，网上有不少，但是几乎都是差不多，很多都还是引用了作者bang自己写的文档的内容，所以我采用一个例子方式进行讲解JSPatch的主要运行流程，其实当然也会引用一些作者的简述，大家可以参照我写的流程讲述，在配合源码或者官方文档的介绍，应该就可以了解JSPatch。

```
 [JPEngine startEngine];
    NSString *sourcePath = [[NSBundle mainBundle] pathForResource:@"demo" ofType:@"js"];
    NSString *script = [NSString stringWithContentsOfFile:sourcePath encoding:NSUTF8StringEncoding error:nil];
    [JPEngine evaluateScript:script];
```
首先是运行[JPEngine startEngine]启动JSPatch，启动过程分为一下两部分：

 * 通过JSContext，声明了一些JS方法到内存，这样我们之后就可以在JS中调用这些方法，主要常用到的包括以下几个方法，同时会监听一个内存告警的通知。
 * 加载JSPatch.js文件，JSPatch文件的主要内容在于定义一些我们之后会用在的JS函数，数据结构以及变量等信息，之后我会在用到的时候详细介绍。
 

```
     context["@_OC_defineClass"] = ^(NSString *classDeclaration, JSValue      *instanceMethods, JSValue *classMethods) {
        return defineClass(classDeclaration, instanceMethods, classMethods);
    };
    
    context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };
    
    context[@"_OC_formatJSToOC"] = ^id(JSValue *obj) {
        return formatJSToOC(obj);
    };
    
    context[@"_OC_formatOCToJS"] = ^id(JSValue *obj) {
        return formatOCToJS([obj toObject]);
    };
    
    context[@"_OC_getCustomProps"] = ^id(JSValue *obj) {
        id realObj = formatJSToOC(obj);
        return objc_getAssociatedObject(realObj, kPropAssociatedObjectKey);
       };
   
    context[@"_OC_setCustomProps"] = ^(JSValue *obj, JSValue *val) {
        id realObj = formatJSToOC(obj);
        objc_setAssociatedObject(realObj, kPropAssociatedObjectKey, val,
        OBJC_ASSOCIATION_RETAIN_NONATOMIC);
   }
   
```


---

#### 脚本运行
我们定义如下的脚本:

```
require('UIAlertView')
defineClass('AppDelegate',['name', 'age', 'temperatureDatas'],
 {
   testFuncationOne: function(index) {
            self.setName('wuyike')
            self.setAge(21)
            self.setTemperatureDatas(new Array(37.10, 36.78, 36.56))
            var alertView = UIAlertView.alloc().initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles(
                    "title", self.name(), self, "OK", null)
            alertView.show()
   }
  },
  {
    testFuncationTwo: function(datas) {
    var alertView = UIAlertView.alloc().initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles(
                        "title", "wwww", self, "OK", null)
                alertView.show()
    }
  });
```
然后就是执行我们上面说的[JPEngine evaluateScript:script]了，程序开始执行我们的脚本，但是在这之前，JSpatch会对我们的脚本做一些处理，这一步同样也包括两个方面:

* 需要给我们的程序加上try catch的部分代码，主要目的是当我们的JS脚本有错误的时候，可以catch到错误信息
* 将所有的函数都改成通过__c原函数的形式进行调用。

也就是最后我们调用的脚本已经变成如下的形式了:

```
;(function(){try{require('UIAlertView')
defineClass('AppDelegate',['name', 'age', 'temperatureDatas'],
 {
   testFuncationOne: function(index) {
            self.__c("setName")('wuyike')
            self.__c("setAge")(21)
            self.__c("setTemperatureDatas")(new Array(37.10, 36.78, 36.56))
            var alertView = UIAlertView.__c("alloc")().__c("initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles")(
                    "title", self.__c("name")(), self, "OK", null)
            alertView.__c("show")()
   }
  },
  {
            testFuncationTwo: function(datas) {
                var alertView = UIAlertView.__c("alloc")().__c("initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles")(
                        "title", "wwww", self, "OK", null)
                alertView.__c("show")()
            }
  });
```
那么为什么需要用函数__c来替换我们的函数呢，因为JS语法的限制对于没有定义的函数JS是无法调用的，也就是调用UIAlertView.alloc()其实是非法的，因为它采用的并不是消息转发的形式，所以作者原来是想把一个类的所有函数都定义在JS上，也就是如下形式：

```
{
    __clsName: "UIAlertView",
    alloc: function() {…},
    beginAnimations_context: function() {…},
    setAnimationsEnabled: function(){…},
    ...
}
```
但是这种形式就必须要遍历当前类的所有方法，还要循环找父类的方法直到顶层，这种方法直接导致的问题就是内存暴涨，所以是不可行的，所以最后作者采用了消息转发的思想，定义了一个_c的原函数，所有的函数都通过_c来转发，这样就解决了我们的问题。

值得一提的是我们的__c函数就是在我们执行JSPatch.js的时候声明到js里的Object方法里去的，就是下面这个函数，_customMethods里面声明了很多需要追加在Object上的函数。

```
  for (var method in _customMethods) {
    if (_customMethods.hasOwnProperty(method)) {
      Object.defineProperty(Object.prototype, method, {value: _customMethods[method], configurable:false, enumerable: false})
    }
  }
```
---

##### 1. require
调用 require('UIAlertView') 后，就可以直接使用UIAlertView这个变量去调用相应的类方法了，require做的事很简单，就是在JS全局作用域上创建一个同名变量，变量指向一个对象，对象属性 __clsName 保存类名，同时表明这个对象是一个 OC Class。

```
var _require = function(clsName) {
  if (!global[clsName]) {
    global[clsName] = {
      __clsName: clsName
    }
  }
  return global[clsName]
}
```
这样我们在接下来调用UIAlertView.__c()方法的时候系统就不会报错了，因为它已经是JS中一个全局的Object对象了。

```
{
  __clsName: "UIAlertView"
}
```
---

##### 2.defineClass
接下来我们就要执行defineClass函数了

```
 global.defineClass = function(declaration, properties, instMethods, clsMethods) 
```
defineClass函数可接受四个参数：

`字符串:”需要替换或者新增的类名:继承的父类名 <实现的协议1，实现的协议2>”`

`[属性]`

`{实例方法}`

`{类方法}`

当我调用这个函数以后主要是做三件事情：

* 执行_formatDefineMethods方法，主要目的是修改传入的function函数的的格式，以及在原来实现上追加了从OC回调回来的参数解析。
* 然后执行_OC_defineClass方法，也就是调用OC的方法，解析传入类的属性，实例方法，类方法，里面会调用overrideMethod方法，进行method swizzing操作，也就是方法的重定向。
* 最后执行_setupJSMethod方法，在js中通过_ocCls记录类实例方法，类方法。

---
**关于_formatDefineMethods**

_formatDefineMethods方法接收的参数是一个方法列表js对象，加一个新的js空对象

```
var _formatDefineMethods = function(methods, newMethods, realClsName) {
    for (var methodName in methods) {
      if (!(methods[methodName] instanceof Function)) return;
      (function(){
        var originMethod = methods[methodName]
        newMethods[methodName] = [originMethod.length, function() {
          try {
             // 通过OC回调回来执行，获取参数 
            var args = _formatOCToJS(Array.prototype.slice.call(arguments))
            var lastSelf = global.self
            global.self = args[0]
            if (global.self) global.self.__realClsName = realClsName
            // 删除前两个参数：在OC中进行消息转发的时候，前两个参数是self和selector，
            // 我们在实际调用js的具体实现的时候，需要把这两个参数删除。
            args.splice(0,1)
            var ret = originMethod.apply(originMethod, args)
            global.self = lastSelf
            return ret
          } catch(e) {
            _OC_catch(e.message, e.stack)
          }
        }]
      })()
    }
  }
```
可以发现，具体实现是遍历方法列表对象的属性（方法名），然后往js空对象中添加相同的属性，它的值对应的是一个数组，数组的第一个值是方法名对应实现函数的参数个数，第二个值是一个函数（也就是方法的具体实现）。_formatDefineMethods作用，简单的说，它把defineClass中传递过来的js对象进行了修改：

```
原来的形式是：
    {
        testFuncationOne：function（）｛...｝
    ｝
修改之后是：
    {
        testFuncationOne: [argCount, function (){...新的实现}]
    }
```
传递参数个数的目的是，runtime在修复类的时候，无法直接解析原始的js实现函数，那么就不知道参数的个数，特别是在创建新的方法的时候，需要根据参数个数生成方法签名，也就是还原方法名字，所以只能在js端拿到js函数的参数个数，传递到OC端。

```
// js 方法
initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles

// oc 方法
initWithTitle:message:delegate:cancelButtonTitle:otherButtonTitles:

```
---

**关于_OC_defineClass**

1. 使用NSScanner分离classDeclaration，分离成三部分
	* 类名 : className
	* 父类名 : superClassName
	* 实现的协议名 : protocalNames

2. 使用NSClassFromString(className)获得该Class对象。
	* 若该Class对象为nil，则说明JS端要添加一个新的类，使用objc_allocateClassPair与objc_registerClassPair注册一个新的类。
	* 若该Class对象不为nil，则说明JS端要替换一个原本已存在的类

3. 根据从JS端传递来的实例方法与类方法参数，为这个类对象添加/替换实例方法与类方法
	* 添加实例方法时，直接使用上一步得到class对象; 添加类方法时需要调用objc_getMetaClass方法获得元类。
	* 如果要替换的类已经定义了该方法，则直接对该方法替换和实现消息转发。
	* 否则根据以下两种情况进行判断
		* 遍历protocalNames，通过objc_getProtocol方法获得协议对象，再使用protocol_copyMethodDescriptionList来获得协议中方法的type和name。匹配JS中传入的selectorName，获得typeDescription字符串，对该协议方法的实现消息转发。
		* 若不是上述两种情况，则js端请求添加一个新的方法。构造一个typeDescription为”@@:\@*”(返回类型为id，参数值根据JS定义的参数个数来决定。新增方法的返回类型和参数类型只能为id类型，因为在JS端只能定义对象)的IMP。将这个IMP添加到类中。

4. 为该类添加setProp:forKey和getProp:方法，使用objc_getAssociatedObject与objc_setAssociatedObject让JS脚本拥有设置property的能力

5. 返回{className:cls}回JS脚本。

不过其中还包括一个overrideMethod方法，不管是替换方法还是新增方法，都是使用overrideMethod方法。它的目的主要在于进行method swizzing操作，也就是方法的重定向。我们把所有的消息全部都转发到ForwardInvocation函数里去执行(不知道的同学请自行补消息转发机制)，这样做的目的在于，我们可以在NSInvocation中获取到所有的参数，这样就可以实现一个通用的IMP，任意方法任意参数都可以通过这个IMP中转，拿到方法的所有参数回调JS的实现。于是overrideMethod其实就是做了如下这件事情:

具体实现，以替换 UIViewController 的 -viewWillAppear: 方法为例：

1. 把UIViewController的-viewWillAppear:方法通过class_replaceMethod()接口指向_objc_msgForward这是一个全局 IMP，OC 调用方法不存在时都会转发到这个IMP上，这里直接把方法替换成这个IMP，这样调用这个方法时就会走到-forwardInvocation:。

2. 为UIViewController添加-ORIGviewWillAppear:和-_JPviewWillAppear: 两个方法，前者指向原来的IMP实现，后者是新的实现，稍后会在这个实现里回调JS函数。

3. 改写UIViewController的-forwardInvocation: 方法为自定义实现。一旦OC里调用 UIViewController 的-viewWillAppear:方法，经过上面的处理会把这个调用转发到-forwardInvocation:，这时已经组装好了一个NSInvocation，包含了这个调用的参数。在这里把参数从 NSInvocation反解出来，带着参数调用上述新增加的方法 -JPviewWillAppear:，在这个新方法里取到参数传给JS，调用JS的实现函数。整个调用过程就结束了，整个过程图示如下：

![1.pic.jpg](../../../../img/technology/2016-08-20/pic_1.png)

---

**关于_setupJSMethod**

```
if (properties) {
      properties.forEach(function(o){
        _ocCls[className]['props'][o] = 1
        _ocCls[className]['props']['set' + o.substr(0,1).toUpperCase() + o.substr(1)] = 1
      })
  }
  
  var _setupJSMethod = function(className, methods, isInst, realClsName) {
    for (var name in methods) {
      var key = isInst ? 'instMethods': 'clsMethods',
          func = methods[name]
      _ocCls[className][key][name] = _wrapLocalMethod(name, func, realClsName)
    }
  }
```
是最后的一步是把之前所有的方法以及属性放入 _ocCls中保存起来，最后再调用require把类保存到全局变量中。

到这一步为止，我们的JS脚本中的所有对象已经，通过runtime替换到我们的程序中去了，也就是说，剩下的就是如何在我们出触发函数以后，能正确的去执行JS中函数的内容。

##### 3. 对象持有/转换

下面引用作者的一段话:

`require('UIView')这句话在JS全局作用域生成了UIView这个对象，它有个属性叫 __isCls，表示这代表一个OC类。调用UIView这个对象的alloc()方法，会去到_c()函数，在这个函数里判断到调用者_isCls 属性，知道它是代表OC类，把方法名和类名传递给OC完成调用。
调用类方法过程是这样，那实例方法呢？UIView.alloc()会返回一个UIView实例对象给JS，这个OC实例对象在JS是怎样表示的？怎样可以在 JS 拿到这个实例对象后可以直接调用它的实例方法UIView.alloc().init()？`

`对于一个自定义id对象，JavaScriptCore 会把这个自定义对象的指针传给JS，这个对象在JS无法使用，但在回传给OC时，OC可以找到这个对象。对于这个对象生命周期的管理，按我的理解如果JS有变量引用时，这个OC对象引用计数就加1，JS变量的引用释放了就减1，如果OC上没别的持有者，这个OC对象的生命周期就跟着 JS走了，会在JS进行垃圾回收时释放。
传回给JS的变量是这个OC对象的指针，这个指针也可以重新传回OC，要在JS调用这个对象的某个实例方法，根据第2点JS接口的描述，只需在_c()函数里把这个对象指针以及它要调用的方法名传回给OC就行了，现在问题只剩下：怎样在_c()函数里判断调用者是一个OC对象指针？
目前没找到方法判断一个JS对象是否表示 OC 指针，这里的解决方法是在OC把对象返回给JS之前，先把它包装成一个NSDictionary：`

```
static NSDictionary *_wrapObj(id obj) {
    return @{@"__obj": obj};
}
```

让 OC 对象作为这个 NSDictionary 的一个值，这样在 JS 里这个对象就变成：

```
{__obj: [OC Object 对象指针]}
```

这样就可以通过判断对象是否有_obj属性得知这个对象是否表示 OC 对象指针，在_c函数里若判断到调用者有_obj属性，取出这个属性，跟调用的实例方法一起传回给OC，就完成了实例方法的调用。

`但是:`

`JS无法调用 NSMutableArray / NSMutableDictionary / NSMutableString 的方法去修改这些对象的数据，因为这三者都在从OC返回到JS时 JavaScriptCore 把它们转成了JS的Array/Object/String，在返回的时候就脱离了跟原对象的联系，这个转换在JavaScriptCore里是强制进行的，无法选择。`

`若想要在对象返回JS后，回到OC还能调用这个对象的方法，就要阻止JavaScriptCore的转换，唯一的方法就是不直接返回这个对象，而是对这个对象进行封装，JPBoxing 就是做这个事情的。`

`把NSMutableArray/NSMutableDictionary/NSMutableString对象作为JPBoxing的成员保存在JPBoxing实例对象上返回给JS，JS拿到的是JPBoxing对象的指针，再传回给OC时就可以通过对象成员取到原来的NSMutableArray/NSMutableDictionary/NSMutableString对象，类似于装箱/拆箱操作，这样就避免了这些对象被JavaScriptCore转换`。

`实际上只有可变的NSMutableArray/NSMutableDictionary/NSMutableString这三个类有必要调用它的方法去修改对象里的数据，不可变的NSArray/NSDictionary/NSString是没必要这样做的，直接转为JS对应的类型使用起来会更方便，但为了规则简单，JSPatch让NSArray/NSDictionary/NSString也同样以封装的方式返回，避免在调用OC方法返回对象时还需要关心它返回的是可变还是不可变对象。最后整个规则还是挺清晰：NSArray/NSDictionary/NSString 及其子类与其他 NSObject 对象的行为一样，在JS上拿到的都只是其对象指针，可以调用它们的OC方法，若要把这三种对象转为对应的JS类型，使用额外的.toJS()的接口去转换。`

`对于参数和返回值是C指针和 Class 类型的支持同样是用 JPBoxing 封装的方式，把指针和Class作为成员保存在JPBoxing对象上返回给JS，传回OC时再解出来拿到原来的指针和Class，这样JSPatch就支持所有数据类型OC<->JS的互传了。`


##### 4. 类型转换

还是引用作者的一段话:

`JS把要调用的类名/方法名/对象传给OC后，OC调用类/对象相应的方法是通过NSInvocation实现，要能顺利调用到方法并取得返回值，要做两件事：`

1. 取得要调用的 OC 方法各参数类型，把 JS 传来的对象转为要求的类型进行调用。 
2. 根据返回值类型取出返回值，包装为对象传回给 JS。

`例如举例子的来讲view.setAlpha(0.5)，JS传递给OC的是一个NSNumber，OC需要通过要调用OC方法的 NSMethodSignature得知这里参数要的是一个float类型值，于是把NSNumber转为float值再作为参数进行OC方法调用。这里主要处理了int/float/bool等数值类型，并对CGRect/CGRange等类型进行了特殊转换处理。`

##### 5. callSelector
callSelector这个就是我们最后执行函数了！但是在执行这个函数之前，前面还有不少东西。

**关于 _c函数**

```
 __c: function(methodName) {
      var slf = this

      if (slf instanceof Boolean) {
        return function() {
          return false
        }
      }
      if (slf[methodName]) {
        return slf[methodName].bind(slf);
      }

      if (!slf.__obj && !slf.__clsName) {
        throw new Error(slf + '.' + methodName + ' is undefined')
      }
      if (slf.__isSuper && slf.__clsName) {
          slf.__clsName = _OC_superClsName(slf.__obj.__realClsName ? slf.__obj.__realClsName: slf.__clsName);
      }
      var clsName = slf.__clsName
      if (clsName && _ocCls[clsName]) {
        var methodType = slf.__obj ? 'instMethods': 'clsMethods'
        if (_ocCls[clsName][methodType][methodName]) {
          slf.__isSuper = 0;
          return _ocCls[clsName][methodType][methodName].bind(slf)
        }

        if (slf.__obj && _ocCls[clsName]['props'][methodName]) {
          if (!slf.__ocProps) {
            var props = _OC_getCustomProps(slf.__obj)
            if (!props) {
              props = {}
              _OC_setCustomProps(slf.__obj, props)
            }
            slf.__ocProps = props;
          }
          var c = methodName.charCodeAt(3);
          if (methodName.length > 3 && methodName.substr(0,3) == 'set' && c >= 65 && c <= 90) {
            return function(val) {
              var propName = methodName[3].toLowerCase() + methodName.substr(4)
              slf.__ocProps[propName] = val
            }
          } else {
            return function(){ 
              return slf.__ocProps[methodName]
            }
          }
        }
      }

      return function(){
        var args = Array.prototype.slice.call(arguments)
        return _methodFunc(slf.__obj, slf.__clsName, methodName, args, slf.__isSuper)
      }
    }
```
`其实_c函数就是一个消息转发中心，它根据传入的参数，这里可以分为两种类型来讲述:`

* 对于实例方法和类方法，最后会调用_methodFunc方法
* 对于自定义的属性，set和get操作。

`对于自定义的属性，其实它并不会将这些属性真正添加到OC中的对象里去，它只会添加一个_ocProps对象，然后在JS中，通过_ocProps对象来保存我们所有定义的属性，要获取值的只要从这个属性里通过name获取就可以了。`

`对于_methodFunc方法，其实就是将OC方法的名字还原，带上参数，然后转发给类方法或者实例方法处理。`

```
  var _methodFunc = function(instance, clsName, methodName, args, isSuper, isPerformSelector) {
    var selectorName = methodName
    if (!isPerformSelector) {
      methodName = methodName.replace(/__/g, "-")
      selectorName = methodName.replace(/_/g, ":").replace(/-/g, "_")
      var marchArr = selectorName.match(/:/g)
      var numOfArgs = marchArr ? marchArr.length : 0
      if (args.length > numOfArgs) {
        selectorName += ":"
      }
    }
    var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
                         _OC_callC(clsName, selectorName, args)
    return _formatOCToJS(ret)
  }
```

对于callSelector方法来讲:

1. 初始化 
	* 将JS封装的instance对象进行拆装，得到OC的对象；
	* 根据类名与selectorName获得对应的类对象与selector；
	* 通过类对象与selector构造对应的NSMethodSignature签名，再根据签名构造NSInvocation对象，并为invocation对象设置target与Selector
2. 根据方法签名，获悉方法每个参数的实际类型，将JS传递过来的参数进行对应的转换(比如说参数的实际类型为int类型，但是JS只能传递NSNumber对象，需要通过[[jsObj toNumber] intValue]进行转换)。转换后使用setArgument方法为NSInvocation对象设置参数。
3. 执行invoke方法。
4. 通过getReturnValue方法获取到返回值。
5. 根据返回值类型，封装成JS中对应的对象(因为JS并不识别OC对象，所以返回值为OC对象的话需封装成{className:className, obj:obj})返回给JS端。

---

### 总结
好，到现在为止，我们所有的流程就已经走完了，我们的js文件也已经生效了。当然，我所说JSPatch原理只是基础的一部份原理，可以使我们的基本流程可以实现，还有一些复杂的操作功能，还需要再深入的学习，才可以掌握，JSPatch对于学习Runtime也是一个不错的例子，就像Aspects一样，大家可以去好好研究一下。