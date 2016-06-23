---
layout: post
title: 自定义NSNotification消息中心
subtitle:   "App 应用笔记"
date:       2016-06-10 22:31:00
author:     "BCM"
header-img: ""
tags:
    - Object C
    - iOS
---

`原创文章转载请注明出处，谢谢`

---

`端午想好好休息一下，于是就没出去玩了，陪陪家人和妹子`

关于NSNotification大家一定不会陌生，所以这里我就不讲关于NSNotification的具体用法了，网上一搜一大片。NSNotification是基于Observe Design Pattern来实现的，所有Register了这个消息的对象，都会收到由NSNotification发送的消息。它有一个自己的消息中心是一个单例，我们基本通过以下三个函数就可以实现一个基本的Observe Pattern。

```
// register notification
- (void)addObserver:(id)observer
		   selector:(SEL)aSelector
		       name:(nullable NSString *)aName
		     object:(nullable id)anObject;

// post	notification	     
- (void)postNotification:(NSNotification *)notification;

// dealloc reomove notification
- (void)removeObserver:(id)observer;

```

但是系统的NSNotification实在是太臃肿了，想想要在所有的业务里到处散布着上面三行代码，这显然是不合适的。我们最理想的效果是只要存在两个方法register和post，register中可以直接传递一个block来完成收到消息时触发的事件，然后在dealloc中自动调用remove的操作。所以我们不妨先定义一下这两个函数：

```
+ (void)postNotificationNamed:(NSString*)notificationName
                       object:(id)object;

// default one parame in callback                 
+ (void)registerNotificationHandlerForName:(NSString *)notificationName
                                    target:(id)target
                                    action:(void(^)(id))action;                     
                       
```
其实我们主要的任务就是要在register函数中去动态在相应的target中添加一个函数，来执行对应的callback操作。

```
@interface NotificationHelper : NSObject
@end

@implementation NotificationHelper
- (void)justForGetTypeEncoding:(id)object {};
@end

+ (void)postNotificationNamed:(NSString *)notificationName object:(id)object {
  [[NSNotificationCenter defaultCenter] postNotificationName:notificationName
                                                      object:object];
}

+ (void)registerNotificationHandlerForName:(NSString *)notificationName
                                    target:(id)target
                                    action:(void(^)(id))action {
  SEL sel = NSSelectorFromString([NSString stringWithFormat:@"_xxx_notification_%@:",
                                  notificationName]);
  if ([target respondsToSelector:sel]) {
    NSError* error = nil;
    [target aspect_hookSelector:sel
                    withOptions:AspectPositionAfter
                     usingBlock:^(id<AspectInfo>info, NSNotification* notification) {
      action(notification.object);
    }
                          error:&error];
  } else {
    typedef void (^BlockType)(id, NSNotification *);
    BlockType block = ^(id target, NSNotification *notification) {
      action(notification.object);
    };
    BOOL success = class_addMethod([target class],
                                   sel,
                                   imp_implementationWithBlock(block),
            method_getTypeEncoding(class_getInstanceMethod([BMTBusinessUtilHelper class], 
            @selector(justForGetTypeEncoding:))));
  }
  
  [[NSNotificationCenter defaultCenter] addAutoDetachedObserver:target
                                                       selector:sel
                                                           name:notificationName
                                                         object:nil];
}

```
上面的思想其实就是当我们去注册一个消息的时候，我们先去判断目标class中是否已经添加了这个SEL，如果没有，我们就去动态添加这个方法到目标class，如果有了，我们就在已存在的方法中追加这个方法。最后把新创建的方法注册到消息中。大致的思路就是这样，剩下的就是关于remove的操作了，思路是我们通过在调用dealloc的时候追加一个remove操作函数到里面，这样就可以保证当我们的class被dealloc的时候，消息能被正确的remove掉。

```
// not use @selector(dealloc), because ARC
[self aspect_hookSelector:NSSelectorFromString(@"dealloc") 
			  withOptions:AspectPositionBefore
 			   usingBlock:^(id<AspectInfo>info) {
        				action();
    			  } error:&error];
```
`讲到这里，基本上已经实现了我们要讲的一些功能，不过这里其实有一个很大的坑，不知道同学们看到这里已经发现了没有，当我们新添加的函数已经存在时，我们会去在已存在的函数后面追加新的action操作，这个时候考虑到如果一个class中有注册消息的操作，但是这个class又被init了多次，那么就会出现我们新添加的SEL中，会有好几个重复的action操作，如果我们的action操作只是为了更新数据，那么这样的开销几乎没有，但是如果我们的action中会有大量的关于刷新界面或者其他UI的操作，其实是会额外附带不少开销的。所以这个就是上述实现的一个缺点，我现在也没有什么好的办法来优化这个问题，只能在代码中尽量避免这种情况的发生，大家想到好的办法也可以告诉我。`

---

**其实OC中的Observe Pattern可以分为三种：delegate，NSNotification，KVO，[这里详细介绍了这三种方式的区别](https://blog.shinetech.com/2011/06/14/delegation-notification-and-observation/)。准确来说对于model-view之间的数据同步，单纯的数据更新来说KVO感觉更加适合一点，因为一旦它的数据发生变化，自动所有接收消息的地方就能马上收到通知。而NSNotification更佳适合模块间的一些界面交互更新的操作，因为它需要主动去发送消息让所有注册了消息的地方都知道然后去执行对应的操作。所以针对上面提出的一些问题，我之后会在这个地方继续补充，毕竟现在我还没想出一个比较好的方案。**
