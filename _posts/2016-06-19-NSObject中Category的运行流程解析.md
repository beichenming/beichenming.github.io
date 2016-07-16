---
layout: post
title: NSObject中Category的运行流程解析
subtitle:   "Runtime Source Code笔记"
date:       2016-06-19 13:31:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - Object C
    - iOS
---

`原创文章转载请注明出处，谢谢`

---

之前都没有很细致的去了解过Category的运行原理，今天正好是周末所以就仔细了解了一下，网上其实有不少好文都详细介绍了Category的原理，不过我还是有自己的一点理解需要讲的。

`Category是装饰者模式的一种体现，主要用于在不改变原有类的前提下，动态的给这个类添加一些方法，同时可以将一个类的实现拆分成多个独立的源文件，方便管理。但是不建议在Category中添加属性，这里说是不建议的意思是其实可以在Category里添加属性的，不过需要一些处理，我在后面会解释。`

首先我们定义了AClass的一个Category ACategory：

```
// AClass.h
@protocol AClass_protocol <NSObject>

- (void)protocolMethods_ACategory;

@end

@interface AClass : NSObject

@end

@interface AClass(ACategory)<AClass_protocol>

@property (nonatomic) NSString *address;

- (void)instanceMethods_ACategory;

+ (void)classMethods_ACategory;

@end

// AClass.m
@implementation AClass

@end

@implementation AClass (ACategory)

@dynamic address;

- (void)instanceMethods_ACategory {
    NSLog(@"instanceMethods_ACategory");
}

+ (void)classMethods_ACategory {
    NSLog(@"classMethods_ACategory");
}

- (void)protocolMethods_ACategory {
    NSLog(@"protocolMethods_ACategory");
}

@end

```
我们在runtime的源码里可以看到Category的定义：

```
typedef struct category_t {
    const char *name; //类的名字
    struct class_t *cls; //指向的类
    struct method_list_t *instanceMethods; //category中所有给类添加的实例方法的列表
    struct method_list_t *classMethods; //category中所有添加的类方法的列表
    struct protocol_list_t *protocols; //category实现的所有协议的列表
    struct property_list_t *instanceProperties; //category中添加的所有属性
} category_t;
```
然后我们编译一下上面的Category，clang -rewrite-objc AClass.m，我们可以得到AClass.cpp,里面包含了如下的内容：

```
 static struct _category_t _OBJC_$_CATEGORY_AClass_$_ACategory __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"AClass",
	0, // &OBJC_CLASS_$_AClass,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_AClass_$_ACategory,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_CLASS_METHODS_AClass_$_ACategory,
	(const struct _protocol_list_t *)&_OBJC_CATEGORY_PROTOCOLS_$_AClass_$_ACategory,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_AClass_$_ACategory,
};
static void OBJC_CATEGORY_SETUP_$_AClass_$_ACategory(void ) {
	_OBJC_$_CATEGORY_AClass_$_ACategory.cls = &OBJC_CLASS_$_AClass;
}

```
我们之前定义的Category最终会被编译成这个样子，上面的四个Struct都可以找到对应定义：

```
// _OBJC_$_CATEGORY_INSTANCE_METHODS_AClass_$_ACategory
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[2];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_AClass_$_ACategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	2,
	{
	{(struct objc_selector *)"instanceMethods_ACategory", "v16@0:8", (void *)_I_AClass_ACategory_instanceMethods_ACategory},
	{(struct objc_selector *)"protocolMethods_ACategory", "v16@0:8", (void *)_I_AClass_ACategory_protocolMethods_ACategory}}
};

// _OBJC_$_CATEGORY_CLASS_METHODS_AClass_$_ACategory
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_CLASS_METHODS_AClass_$_ACategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"classMethods_ACategory", "v16@0:8", (void *)_C_AClass_ACategory_classMethods_ACategory}}
};

// _OBJC_CATEGORY_PROTOCOLS_$_AClass_$_ACategory
static struct /*_protocol_list_t*/ {
	long protocol_count;  // Note, this is 32/64 bit
	struct _protocol_t *super_protocols[1];
} _OBJC_CATEGORY_PROTOCOLS_$_AClass_$_ACategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	1,
	&_OBJC_PROTOCOL_AClass_protocol
};

// _OBJC_$_PROP_LIST_AClass_$_ACategory
static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_AClass_$_ACategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"address","T@\"NSString\",D,N"}}
};


```

我们再来看一下两个Category的时候编译出来的category_t结构，其实就是多了一个Category添加到数组里:

```
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [2] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
	&_OBJC_$_CATEGORY_AClass_$_ACategory1,
	&_OBJC_$_CATEGORY_AClass_$_ACategory2,
};
```

好了，到现在为止我们正式得到了在runtime之前的Category的结构，接下来就是加载Category了，这里我就不讲整个过程了，网上有很多讲的，具体说一下Category和Class中同名方法的执行顺序的问题：

* Category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果Category和原来类都有methodA，那么Category附加完成之后，类的方法列表里会有两个methodA。
* Category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的Category的方法会“覆盖”掉原来类的同名方法，这是因为runtime时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法就会停止。

会造成上面这个结果的原因主要是因为下面这个函数：

```
static void 
attachMethodLists(class_t *cls, method_list_t **addedLists, int addedCount, 
                  BOOL methodsFromBundle, BOOL *inoutVtablesAffected)
{
    rwlock_assert_writing(&runtimeLock);

    // Don't scan redundantly
    BOOL scanForCustomRR = !UseGC && !cls->hasCustomRR();

    // Method list array is NULL-terminated.
    // Some elements of lists are NULL; we must filter them out.

    method_list_t **oldLists = cls->data()->methods;
    int oldCount = 0;
    if (oldLists) {
        while (oldLists[oldCount]) oldCount++;
    }
        
    int newCount = oldCount + 1;  // including NULL terminator
    for (int i = 0; i < addedCount; i++) {
        if (addedLists[i]) newCount++;  // only non-NULL entries get added
    }

    method_list_t **newLists = (method_list_t **)
        _malloc_internal(newCount * sizeof(*newLists));

    // Add method lists to array.
    // Reallocate un-fixed method lists.
    // The new methods are PREPENDED to the method list array.

    newCount = 0;
    int i;
    for (i = 0; i < addedCount; i++) {
        method_list_t *mlist = addedLists[i];
        if (!mlist) continue;

        // Fixup selectors if necessary
        if (!isMethodListFixedUp(mlist)) {
            mlist = fixupMethodList(mlist, methodsFromBundle);
        }

        // Scan for vtable updates
        if (inoutVtablesAffected  &&  !*inoutVtablesAffected) {
            uint32_t m;
            for (m = 0; m < mlist->count; m++) {
                SEL sel = method_list_nth(mlist, m)->name;
                if (vtable_containsSelector(sel)) {
                    *inoutVtablesAffected = YES;
                    break;
                }
            }
        }

        // Scan for method implementations tracked by the class's flags
        if (scanForCustomRR) {
            uint32_t m;
            for (m = 0; m < mlist->count; m++) {
                SEL sel = method_list_nth(mlist, m)->name;
                if (isRRSelector(sel)) {
                    cls->setHasCustomRR();
                    scanForCustomRR = NO;
                    break;
                }
            }
        }
        
        // Fill method list array
        newLists[newCount++] = mlist;
    }

    // Copy old methods to the method list array
    for (i = 0; i < oldCount; i++) {
        newLists[newCount++] = oldLists[i];
    }
    if (oldLists) free(oldLists);

    // NULL-terminate
    newLists[newCount++] = NULL;
    cls->data()->methods = newLists;
}
```

`这个函数的主要思想是重新创建一个方法列表，然后先添加newList，然后再添加oldList，另外由于class的优先级要高于category，所以category的同名方法肯定是优于class中执行的，而相同category中的同名方法就要看编译顺序了。关于cateory中load方法我已经在之前有提到过了，所以就不讲了。`

**好说到这里，关于Category的运行流程已经讲完了，剩下来的就是关于在Category中定义属性的问题了，其实是不建议在Category中定义属性的，Effective Object C2.0中的26条也说了这件事，勿在分类中声明属性；由于属性在Category中不能自动生成get和set方法，所以我们需要先把存取方法声明成@dynamic，然后在runtime的时候去实现它的get和set方法，通过objc_getAssociatedObject和objc_setAssociatedObject来完成，这样就可以实现在Category中定义属性了。但是这是个不建议的做法，容易在内存管理上出现一些难排查的问题。**