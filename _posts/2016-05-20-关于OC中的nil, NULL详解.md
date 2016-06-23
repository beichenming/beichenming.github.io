---
layout: post
title: 关于OC中的nil, NULL详解
subtitle:   "Object C 基础知识笔记"
date:       2016-05-20 21:31:00
author:     "BCM"
header-img: ""
tags:
    - Object C
    - iOS
---

`原创文章转载请注明出处，谢谢`

---

### 关于OC中的nil, NULL详解
**我相信很多同学对于这四种表示空的方式肯定不会陌生了，网上也有不少介绍四种方式区别的，不过我还是想说一些自己的理解。**

#### 关于nil
**nil的定义是null pointer to object-c object，指的是一个OC对象指针为空，本质就是(id)0，是OC对象的字面0值**

`不过这里有必要提一点就是OC中给空指针发消息不会崩溃的语言特性，原因是OC的函数调用都是通过objc_msgSend进行消息发送来实现的，相对于C和C++来说，对于空指针的操作会引起Crash的问题，而objc_msgSend会通过判断self来决定是否发送消息，如果self为nil，那么selector也会为空，直接返回，所以不会出现问题。`

`这里补充一点，如果一个对象已经被释放了，那么这个时候再去调用方法肯定是会Crash的，因为这个时候这个对象就是一个野指针了，安全的做法是释放后将对象重新置为nil，使它成为一个空指针，大家可以在关闭ARC后手动release对象验证一下。(2016.05.24补)`

```
	NSString *name = @"Allen";
	
	if (name != nil && [name isEqualToString:@"Allen"]) {
	    NSLog(@"name: %@", name);
	} else {
	    NSLog(@"name is nil");
	}
	
	//or
	if ([name isEqualToString:@"Allen"]) {
	    NSLog(@"name: %@", name);
	} else {
	    NSLog(@"name is nil");
	}
```
**上面的两种判断都是正确的，我们不必担心当name为nil时调用isEqualToString会出现Crash，但是我还是想说，在使用一个对象之前判断它是否为nil是一个很好的习惯，个人觉得有两个原因:**

1. `降低时间复杂度（感觉可以这么说吧），如果你增加了nil的判断，那么不需要对空指针发送消息了，发消息其实是件费时的操作。`***[详情可以看这里](http://xuzhe.com/?p=630)***
2. `把判断为空养成习惯其实是好事，这样在你切换语言时也不容易出错。`

#### 关于NULL
**NULL的定义是null pointer to primitive type or absence of data，指的是一般的基础数据类型为空，可以给任意的指针赋值。本质就是(void *)0，是C指针的字面0值。**

```
  NSInteger *pointerA = NULL;
  NSInteger pointerB = 10;
  pointerA = &pointerB;
  NSLog(@"%ld", *pointerA);

```
`我们要尽量不去将NULL初始化OC对象，可能会产生一些异常的错误，要使用nil，NULL主要针对基础数据类型。`

#### 关于Nil
**Nil的定义是null pointer to object-c class，指的是一个类指针为空。本质就是(class)0，OC类的字面零值。**

```
	Class class = [NSString class];
	if (class != Nil) {
	    NSLog(@"class name: %@", class);
	}
```

#### 关于NSNull
**NSNull好像没有什么具体的定义（懵），它包含了唯一一个方法+(NSNull*)null，[NSNull null]是一个对象，用来表示零值的单独的对象。**

```
   NSMutableDictionary *dictionary = [[NSMutableDictionary alloc] init];
   NSString *nameOne = @"Allen";
   NSString *nameTwo = [NSNull null]; //not use nil
   NSString *nameThree = @"Tom";
   [dictionary setObject:nameOne forKey:@"nameOne"];
   [dictionary setObject:nameTwo forKey:@"nameTwo"];
   [dictionary setObject:nameThree forKey:@"nameThree"];
   NSLog(@"names: %@", dictionary);
   
   NSMutableArray *array = [[NSMutableArray alloc] init];
  [array addObject:nameOne];
  [array addObject:nameTwo];
  [array addObject:nameThree];
  NSLog(@"names : %@", array);
```
`NSNull主要用在不能使用nil的场景下，比如NSMutableArray是以nil作为数组结尾判断的，所以如果想插入一个空的对象就不能使用nil，NSMutableDictionary也是类似，我们不能使用nil作为一个object，而要使用NSNull`

### 总结
**其实这几种空类型还是很好理解的，重要的是我们需要在平时的项目中也切实运用起来，不小心初始化的错误可能导致一些难以发现的Bug。**