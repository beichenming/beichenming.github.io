---
layout: post
title: NSObject中+(void)load运行流程解析
subtitle:   "Runtime Source Code笔记"
date:       2016-06-05 21:00:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - Object C
    - iOS
---

`原创文章转载请注明出处，谢谢`

---

`两个星期没有更新文章了，实属没有时间的赶脚，确实将自己知道的一些东西整理出来是要比直接讲给别人听要麻烦很多的，好，言归正传。`

#### +(void)load

首先官方的spec里是这么定义的load函数的：

```
Discussion
The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.

The order of initialization is as follows:

1.All initializers in any framework you link to.

2.All +load methods in your image.(这个image指的就是二进制可执行文件)

3.All C++ static initializers and C/C++ __attribute__(constructor) functions in your image.

4.All initializers in frameworks that link to you.

In addition:

1.A class’s +load method is called after all of its superclasses’ +load methods.

2.A category +load method is called after the class’s own +load method.

In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.
```
由此我们可以知道load函数是只要你动态加载或者静态引用了这个类，那么load就会被执行，它并不需要你显示的去创建一个类后才会执行，同时只执行一次。

另外就是关于load的执行顺序问题，所有的superclass的load执行完以后才会执行该类的load，以及class中的load方法是先于category中的load执行的。

`这里还有关于category多个load方法执行的先后问题,我们到具体讲category的时候再来细说`

接下来我们具体看一下+(void)load方法是如何被执行的：

![Paste_Image.png](../../../../img/technology/2016-06-05/pic_one.png)

我们可以看到，在执行到+(void)load之前的堆栈信息是这样的，我们可以从堆栈中看出dyld start，然后进入dyld main函数的入口，之后就开始初始化一系列方法，通过ImageLoader将二进制的文件加载进来，最后通知runtime开始执行，以上是我刚开始看到时候的猜想，那么到底是不是这样呢，我们来验证一下。其实dyld也是开源的[(dynamic link editor，动态链接器)](https://github.com/opensource-apple/dyld)，但是我并没有去看过，因为水平不够嘛，不过未来还是会去看看的。言归正传，这个时候我们看到ImageLoader结束以后开始执行了我们的第一个runtime函数：load_images（其实不是第一个，后面会改正），我们可以来看看load_images干了些什么事：

```

/***********************************************************************
* load_images
* Process +load in the given images which are being mapped in by dyld.
* Calls ABI-agnostic code after taking ABI-specific locks.
*
* Locking: write-locks runtimeLock and loadMethodLock
**********************************************************************/
PRIVATE_EXTERN const char *
load_images(enum dyld_image_states state, uint32_t infoCount,
            const struct dyld_image_info infoList[])
{
    BOOL found;

    recursive_mutex_lock(&loadMethodLock);

    // Discover load methods
    rwlock_write(&runtimeLock);
    found = load_images_nolock(state, infoCount, infoList);
    rwlock_unlock_write(&runtimeLock);

    // Call +load methods (without runtimeLock - re-entrant)
    if (found) {
        call_load_methods();
    }

    recursive_mutex_unlock(&loadMethodLock);

    return NULL;
}

```
我们观察一下，这个函数的loadMethodLock都有lock和unlock（很多函数都有这些操作），我猜这就是＋(void)load为什么是线程安全的原因，load_images很显然主要是去调用了一个load_images_nolock和call_load_methods的两个函数，如果load_images_nolock返回的是true我们才会去执行call_load_methods，那我们来看看load_images_nolock里到底干了些什么：

```
/***********************************************************************
* load_images_nolock
* Prepares +load in the given images which are being mapped in by dyld.
* Returns YES if there are now +load methods to be called by call_load_methods.
*
* Locking: loadMethodLock(both) and runtimeLock(new) acquired by load_images
**********************************************************************/
PRIVATE_EXTERN BOOL 
load_images_nolock(enum dyld_image_states state,uint32_t infoCount,
                   const struct dyld_image_info infoList[])
{
    BOOL found = NO;
    uint32_t i;

    i = infoCount;
    while (i--) {
        header_info *hi;
        for (hi = FirstHeader; hi != NULL; hi = hi->next) {
            const headerType *mhdr = (headerType*)infoList[i].imageLoadAddress;
            if (hi->mhdr == mhdr) {
                prepare_load_methods(hi);
                found = YES;
            }
        }
    }

    return found;
}

```
我们通过注释也能明白，它其实就是去查找load这个方法，如果有就通知call_load_methods，那么prepare_load_methods又是个什么鬼呢，我们继续看：

```
PRIVATE_EXTERN void prepare_load_methods(header_info *hi)
{
    size_t count, i;

    rwlock_assert_writing(&runtimeLock);

    class_t **classlist = 
        _getObjc2NonlazyClassList(hi, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(hi, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        // Do NOT use cat->cls! It may have been remapped.
        class_t *cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(isRealized(cls->isa));
        add_category_to_loadable_list((Category)cat);
    }
}

```
prepare_load_methods其实就是为load方法做准备，我们可以看到class的load是优先于category的load执行的，原因就是class的load会先添加到list里面，之后才回添加category里的load方法，这里我们看看schedule_class_load函数干了些什么：

```
/***********************************************************************
* prepare_load_methods
* Schedule +load for classes in this image, any un-+load-ed 
* superclasses in other images, and any categories in this image.
**********************************************************************/
// Recursively schedule +load for cls and any un-+load-ed superclasses.
// cls must already be connected.
static void schedule_class_load(class_t *cls)
{
    if (!cls) return;
    assert(isRealized(cls));  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(getSuperclass(cls));

    add_class_to_loadable_list((Class)cls);
    changeInfo(cls, RW_LOADED, 0); 
}
```
从这里我们可以看到，子类是需要先将superclass的load方法加载add进列表后才会去添加subclass的load方法，一个递归的过程，所以这就是为什么父类的load会优先于子类执行，同时也说明了子类中为什么不需要显示调用[super load]的原因。然后就是add_class_to_loadable_list和add_category_to_loadable_list两个方法了，但是我暂时不懂多个category中load的执行顺序，因为看上去像是直接按照for循环的顺序来的，等我以后明白了就会补在讲category的文章里。

说到这里，我们又要回到最前面了call_load_methods函数，我们再来看看：

```
/***********************************************************************
* call_load_methods
* Call all pending class and category +load methods.
* Class +load methods are called superclass-first. 
* Category +load methods are not called until after the parent class's +load.
* 
* This method must be RE-ENTRANT, because a +load could trigger 
* more image mapping. In addition, the superclass-first ordering 
* must be preserved in the face of re-entrant calls. Therefore, 
* only the OUTERMOST call of this function will do anything, and 
* that call will handle all loadable classes, even those generated 
* while it was running.
*
* The sequence below preserves +load ordering in the face of 
* image loading during a +load, and make sure that no 
* +load method is forgotten because it was added during 
* a +load call.
* Sequence:
* 1. Repeatedly call class +loads until there aren't any more
* 2. Call category +loads ONCE.
* 3. Run more +loads if:
*    (a) there are more classes to load, OR
*    (b) there are some potential category +loads that have 
*        still never been attempted.
* Category +loads are only run once to ensure "parent class first" 
* ordering, even if a category +load triggers a new loadable class 
* and a new loadable category attached to that class. 
*
* Locking: loadMethodLock must be held by the caller 
*   All other locks must not be held.
**********************************************************************/
PRIVATE_EXTERN void call_load_methods(void)
{
    static BOOL loading = NO;
    BOOL more_categories;

    recursive_mutex_assert_locked(&loadMethodLock);

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    loading = NO;
}

```
这里就是去具体执行load函数了，分为执行class的load函数和category的load函数，两个函数有点复杂具体不细说。

现在到这里和我们的堆栈信息都是一样，不过这里就有个问题了，load_images这个函数到底是怎么开始执行的，换句话说就是是谁开始调用了它呢，所以我们就有必要来看一下了，搜索一下runtime中调用load_images的地方，只有一个叫做_objc_init的地方，我们来看一下函数：

```
/***********************************************************************
* _objc_init
* Static initializer. Registers our image notifier with dyld.
**********************************************************************/
static __attribute__((constructor))
void _objc_init(void)
{
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    lock_init();
    exception_init();
        
    // Register for unmap first, in case some +load unmaps something
    _dyld_register_func_for_remove_image(&unmap_image);
    dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_images);
    dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}
```
这个函数一看就是runtime加载的入口函数，通过map_image把image加载到内存里，事实是否如此，我们就来验证一下好了，断点是最能证明的。

![Paste_Image.png](../../../../img/technology/2016-06-05/pic_two.png)

果然就像我们说的，最后附上一张所以讲到的函数断点图，同学们可以自己验证一下。

![Paste_Image.png](../../../../img/technology/2016-06-05/pic_three.png)

`最后就是讲关于+(void)load函数的具体应用了，load函数其实是在main函数之前就回被调用，所以Method Swizzing肯定就是在这个时候调用的，同时比如在运行时动态添加属性，方法甚至类等等，但是千万不要在这里初始化OC的对象，因为load执行的时候你不知道你使用的对象是否已经被加载进来，所以无法预知情况。`

---

**最后扯一句废话，写博客其实就是一个自己记录知识的过程，我以前习惯用笔记记，有那么像圣经一样的厚厚的一本，现在回过头来有一半基本都已经忘了，但是自从我写博客记录以来，所有的东西都是记的印象非常深刻，因为这都是自己一句一句码出来的，都是一遍一遍错别字校验来的，所以你自己会有很深的印象，所以如果只是抄别人的博客，那就没什么意思了。**
