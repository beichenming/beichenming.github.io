---
layout: post
title: NSObject中+(void)initialize运行流程解析
subtitle:   "Runtime Source Code笔记"
date:       2016-06-14 21:00:00
author:     "BCM"
header-img: ""
tags:
    - Object C
    - iOS
---

`原创文章转载请注明出处，谢谢`

上次讲了关于＋(void)load的运行流程解析，所以肯定就有必要来讲一下+(void)initialize，因为大家都会拿这两个函数做比较。

`+(void)initialize`

官方的spec是这么定义的：

```
The runtime sends initialize to each class in a program just before the class, or any class that inherits from it, is sent its first message from within the program. The runtime sends the initialize message to classes in a thread-safe manner. Superclasses receive this message before their subclasses. The superclass implementation may be called multiple times if subclasses do not implement initialize—the runtime will call the inherited implementation—or if subclasses explicitly call [super initialize]. If you want to protect yourself from being run multiple times, you can structure your implementation along these lines:

+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}

Because initialize is called in a thread-safe manner and the order of initialize being called on different classes is not guaranteed, it’s important to do the minimum amount of work necessary in initialize methods. Specifically, any code that takes locks that might be required by other classes in their initialize methods is liable to lead to deadlocks. Therefore you should not rely on initialize for complex initialization, and should instead limit it to straightforward, class local initialization.

Special Considerations
initialize is invoked only once per class. If you want to perform independent initialization for the class and for categories of the class, you should implement load methods.
```
由此我们可以知道以下几点：

* 每个类的initialize只会执行一次，在alloc之前
* initialize是线程安全的
* 父类的调用的会先于子类调用
* 父类的调用会被执行多次，如果子类的initialize函数没有被实现
* 不需要显示[super initialize]的调用

那么initialize在我们的runtime中到底是怎么执行的呢？

`我们通过断点会发现在initialize执行之前会有一个_class_initialize的函数调用，通过runtime的源码就会发现有一个_class_initialize的函数。`

```
/***********************************************************************
* class_initialize.  Send the '+initialize' message on demand to any
* uninitialized class. Force initialization of superclasses first.
*
* Called only from _class_lookupMethodAndLoadCache (or itself).
**********************************************************************/
PRIVATE_EXTERN void _class_initialize(Class cls)
{
    Class supercls;
    BOOL reallyInitialize = NO;

    // Get the real class from the metaclass. The superclass chain 
    // hangs off the real class only.
    cls = _class_getNonMetaClass(cls);

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = _class_getSuperclass(cls);
    if (supercls  &&  !_class_isInitialized(supercls)) {
        _class_initialize(supercls);
    }
    
    // Try to atomically set CLS_INITIALIZING.
    monitor_enter(&classInitLock);
    if (!_class_isInitialized(cls) && !_class_isInitializing(cls)) {
        _class_setInitializing(cls);
        reallyInitialize = YES;
    }
    monitor_exit(&classInitLock);
    
    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.
        
        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);
        
        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if 
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: calling +[%s initialize]",
                         _class_getName(cls));
        }

        ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);

        if (PrintInitializing) {
            _objc_inform("INITIALIZE: finished +[%s initialize]",
                         _class_getName(cls));
        }        
        
        // Done initializing. 
        // If the superclass is also done initializing, then update 
        //   the info bits and notify waiting threads.
        // If not, update them later. (This can happen if this +initialize 
        //   was itself triggered from inside a superclass +initialize.)
        
        monitor_enter(&classInitLock);
        if (!supercls  ||  _class_isInitialized(supercls)) {
            _finishInitializing(cls, supercls);
        } else {
            _finishInitializingAfter(cls, supercls);
        }
        monitor_exit(&classInitLock);
        return;
    }
    
    else if (_class_isInitializing(cls)) {
        // We couldn't set INITIALIZING because INITIALIZING was already set.
        // If this thread set it earlier, continue normally.
        // If some other thread set it, block until initialize is done.
        // It's ok if INITIALIZING changes to INITIALIZED while we're here, 
        //   because we safely check for INITIALIZED inside the lock 
        //   before blocking.
        if (_thisThreadIsInitializingClass(cls)) {
            return;
        } else {
            monitor_enter(&classInitLock);
            while (!_class_isInitialized(cls)) {
                monitor_wait(&classInitLock);
            }
            monitor_exit(&classInitLock);
            return;
        }
    }
    
    else if (_class_isInitialized(cls)) {
        // Set CLS_INITIALIZING failed because someone else already 
        //   initialized the class. Continue normally.
        // NOTE this check must come AFTER the ISINITIALIZING case.
        // Otherwise: Another thread is initializing this class. ISINITIALIZED 
        //   is false. Skip this clause. Then the other thread finishes 
        //   initialization and sets INITIALIZING=no and INITIALIZED=yes. 
        //   Skip the ISINITIALIZING clause. Die horribly.
        return;
    }
    
    else {
        // We shouldn't be here. 
        _objc_fatal("thread-safe class init in objc runtime is buggy!");
    }
}
```
我们观察这个函数发现：

```
    supercls = _class_getSuperclass(cls);
    if (supercls  &&  !_class_isInitialized(supercls)) {
        _class_initialize(supercls);
    }
```
`它会去找父类的initialize先执行，然后再去执行子类的，一个递归的过程，这就解释了为什么父类的调用的会先于子类调用。`

```
 ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
```
`这段代码很重要，说明了initialize的调用也是和其他函数一样是通过objc_msgSend的方式来执行的，这和load函数的区别在于，load是记录了一个自己的方法列表，依次去执行所有已有的方法，所以即使子类没有实现load方法，它也不会去调用父类的方法；但是initialize是如果子类没有实现initialize方法，那么它就会调用父类的实现，这个证明了上面提到的父类的调用会被执行多次，如果子类的initialize函数没有被实现的情况下，而且如果一个cateory实现了initialize方法，那么它就会覆盖原来类里的initialize方法（同名方法覆盖，只取method_list里的最上层的方法）。最后就是由于父类可能会被执行多次，所以为了避免多次执行可能带来的危害，我们可以使用spec上的做法：`

```
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```
引用一下网上对于load和initialize的区别总结：

```
			  +load				 +initialize
调用时机		被添加到runtime时	 收到第一条消息前，可能永远不调用

调用顺序		父类->子类->分类			父类->子类

调用次数	    1次					多次

是否需要显式     否                      否
调用父类实现				

是否沿用         否                      是
父类的实现						

分类中的实现      类和分类都执行	        覆盖类中的方法，只执行分类的实现
```