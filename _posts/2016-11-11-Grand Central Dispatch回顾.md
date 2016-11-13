---
layout: post
title: Grand Central Dispatch回顾
subtitle:   "Object C 知识笔记"
date:       2016-11-11 21:31:00
author:     "BCM"
header-img: "../../img/post-bg-2015.jpg"
tags:
    - Object C
    - iOS
---

**重温了一遍关于GCD方面的一些知识，于是重新整理了一下。**

#### 关于Dispatch Queue
**Dispatch Queue可以分为两种队列，一种是等待现在执行中处理的Serial Dispatch Queue，即串行执行；另一种是不等待现在执行中处理的Concurrent Dispatch Queue，即并行执行；**

`Concurrent Dispatch Queue并行执行的处理数是由CPU核数，CPU负荷以及Dispatch Queue中的处理所决定。`

#### 关于dispatch-queue-create

**dispatch-queue-create用于创建Dispatch Queue，可以创建Serial Dispatch Queue和Concurrent Dispatch Queue两种。**

在《Objective-C高级编程iOS与OSX多线程和内存管理》一书中说到dispatch-queue-create不受ARC控制，需要我们自己手动disaptch-retain和dispatch-release，这个其实是不对的。在官方的文档中已经说明在OSX10.8和iOS10.6以后，ARC已经支持自动管理Dispatch Queue的创建了，不要我们手动release和retain了；但是如果你需要在开启ARC的情况下同时手动retian/release，那么就需要在compiler flags设置-DOS-OBJECT-USE-OBJC = 0。

#### 关于Main Disaptch Queue和Global Disaptch Queue

**Main Disaptch Queue和Global Disaptch Queue是两个系统的标准Dispatch Queue。**

Main Disaptch Queue是在主线程中执行的Dispatch Queue，因为主线程只有一个，所以Main Dispatch Queue就是Serial Dispatch Queue。

Global Disaptch Queue是所有线程都可以使用的Concurrent Dispatch Queue，Global Disaptch Queue有四个执行优先级：

* High Priority（最高优先级）
* Default Priority（默认优先级）
* Low Priority（低优先级）
* Background Priority（后台优先级）

`但是通过XNU内核用于Global Disaptch Queue的线程并不能保持实时性，因此执行优先级只是大致的判断。Global Disaptch Queue的默认执行优先级是Default Priority。`

```
/*
 * Main Dispatch Queue
 */
 dispatch_queue_t mainDispatchQueue = dispatch_get_main_queue();
 
 /*
  * Global Dispatch Queue(High Priority)
  */
 dispatch_queue_t globalDispatchQueueHigh = 
 	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
 
 /*
  * Global Dispatch Queue(Default Priority)
  */
 dispatch_queue_t globalDispatchQueueDefault = 
 	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
  /*
  * Global Dispatch Queue(Low Priority)
  */
 dispatch_queue_t globalDispatchQueueLow = 
 	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
 
  /*
  * Global Dispatch Queue(Background Priority)
  */
 dispatch_queue_t globalDispatchQueueBackground = 
 	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
 
```

#### 关于dispatch-set-target-queue

dispatch-set-target-queue一共有两个作用。

**作用一：修改Dispatch Queue的执行优先级；通过dispatch-queue-create函数生成的Dispatch Queue默认优先级都是Default Priority。**

```
/*
 *修改serialQueue优先级至Background Priority
 */
dispatch_queue_t serialQueue = dispatch_queue_create("com.example.gcd.serialQueue", NULL);  
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);  
dispatch_set_target_queue(serialQueue, globalQueue); 
```

注意不要修改Main Disaptch Queue和Global Disaptch Queue的优先级，因为这种情况是不可预知的。

**作用二：修改用户队列的目标队列，使多个Serial Dispatch Queue在目标Queue只能同时执行一个处理，防止并行执行。**
  
```
dispatch_queue_t serialQueue = dispatch_queue_create("com.example.gcd.serialQueue", DISPATCH_QUEUE_SERIAL);  
      
dispatch_queue_t queue1 = dispatch_queue_create("com.example.gcd.queue1", DISPATCH_QUEUE_SERIAL); 
dispatch_queue_t queue2 = dispatch_queue_create("com.example.gcd.queue2", DISPATCH_QUEUE_SERIAL);  
dispatch_queue_t queue3 = dispatch_queue_create("com.example.gcd.queue3", DISPATCH_QUEUE_SERIAL);  
      
dispatch_set_target_queue(queue1, serialQueue);  
dispatch_set_target_queue(queue2, serialQueue);  
dispatch_set_target_queue(queue3, serialQueue);  
      
      
dispatch_async(queue1, ^{  
    NSLog(@"queue1-start");
    sleep(1.0f);
    NSLog(@"queue1-end");
});  
  
dispatch_async(queue2, ^{  
    NSLog(@"queue2-start");
    sleep(1.0f);
    NSLog(@"queue2-end");
});  
dispatch_async(queue3, ^{  
	NSLog(@"queue3-start");
	sleep(1.0f);
	NSLog(@"queue3-end");
});  

//out put  多个Serial Queue并发执行，每次只能执行一个serial Queue的内容
queue1-start
queue1-end
queue2-start
queue2-end
queue3-start
queue3-end

```

#### 关于dispatch-after和dispatch-once

**dispatch-after函数并不是在指定时间后执行处理，而是在指定时间追加处理时间到Dispatch queue后再进行执行。**

关于dispatch-time-t的类型可以由dispatch-time和dispatch-walltime两个函数来生成。

dispatch-time函数能够获取从第一个参数dispatch-time-t类型值中指定时间开始，到第二个参数指定的毫微秒单位时间后的时间。此时间是指相对时间。

```
// 延时一秒以后
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
```

dispatch-walltime函数能够获取从第一个参数struct timespec结构体时间开始，此时间是指绝对时间。

struct timespec类型的时间可以通过NSDate类对象转换而成。

```
NSDate *date = [NSDate date];
NSTimeInterval interval;
double second, subsecond;
struct timespec time;
    
interval = [date timeIntervalSince1970];
subsecond = modf(interval, &second);
time.tv_sec = second;
time.tv_nsec = subsecond * NSEC_PER_SEC;
dispatch_time_t milestone milestone = dispatch_walltime(&time, 0);

```

dispatch-once函数的目的是保证在应用程序中执行中只执行指定处理，经常出现在单例的初始化里面，通过disptach-once函数，即使在多线程环境下执行也是安全的。

```
static dispatch_once_t once;

dispatch_once(&once, ^{
	// init class
});
```

#### 关于Dispatch Group的作用

**Dispatch Group的作用是当队列中的所有任务都执行完毕后在去做一些操作，主要针对Concurrent Dispatch Queue中多个处理结束后追加的操作。**

Dispatch Group分为两种方式可以实现上面需求。

**第一种是使用dispatch-group-async函数，将队列与任务组进行关联并自动执行队列中的任务。**

```
    dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"blk1");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"blk2");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"blk3");
    });
    
    dispatch_group_notify(group, queue, ^{
        NSLog(@"done");
    });

	// output
	blk3
	blk2
	blk1
	done

```

dispatch-group-async函数会将队列与相应的任务组进行关联同时自动执行，当与任务组关联的队列中的任务都执行完毕后，会通过dispatch-group-notify函数发出通知告诉用户任务组中的所有任务都执行完毕了，有点类似于dispatch-barrier-async，另外dispatch-group-notify方式并不会阻塞线程。

但是如果我们使用dispatch-group-wait函数，那么就会阻塞当前线程，等待全部处理执行结束。

```
    dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"blk1");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"blk2");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"blk3");
    });
    
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    NSLog(@"done");
    
	//output
	blk2
	blk3
	blk1
	done

```
dispatch-group-wait函数的返回值不为0就意味着虽然经过了指定的时间，但是属于Dispatch Group的某一个处理还在执行中，如果返回值为0为全部执行结束，当等待时间为DISPATCH-TIME-FOREVER，由dispatch-group-wait函数返回时，由于属于Dispatch Group的处理必定全部执行结束，因此返回值一直为0。当指定为DISPATCH-TIME-NOW则不用任何等待即可判定属于Dispatch Group的处理是否执行结束。


**第二种是使用手动的将队列与组进行关联然后使用异步将队列进行执行，也就是dispatch-group-enter与dispatch-group-leave方法的使用。dispatch-group-enter函数进入到任务组中，然后异步执行队列中的任务，最后使用dispatch-group-leave函数离开任务组即可。**

```
    dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"blk1");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"blk2");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"blk3");
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, queue, ^{
        NSLog(@"done");
    });
    
    // output
    blk2
	 blk3
	 blk1
	 done
```
dispatch-group-enter和dispatch-group-leave必须同时匹配出现才可以，不然就会出现无法预知的情况。

#### 关于dispatch-barrier-async

**dispatch-barrier-async函数的目的基本就是为了读写锁的问题。**

`对于Concurrent Dispatch Queue可能会所产生的数据库同时读写的问题，使用dispatch-barrier-async就可以很好的避免这个问题。`

```
    dispatch_queue_t queue = dispatch_queue_create("com.example.gcd.queue", DISPATCH_QUEUE_CONCURRENT);
    __block NSInteger index = 0;
    dispatch_async(queue, ^{NSLog(@"blk0_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk1_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk2_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk3_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk4_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk5_for_reading index %ld", index);});
    
    dispatch_barrier_sync(queue, ^{
        NSLog(@"----------------------------------");
        index++;
    });
    
    dispatch_async(queue, ^{NSLog(@"blk6_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk7_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk8_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk9_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk10_for_reading index %ld", index);});
    dispatch_async(queue, ^{NSLog(@"blk11_for_reading index %ld", index);});
    
//output
blk0_for_reading index 0
blk1_for_reading index 0
blk2_for_reading index 0
blk3_for_reading index 0
blk5_for_reading index 0
blk4_for_reading index 0
----------------------------------
blk6_for_reading index 1
blk7_for_reading index 1
blk8_for_reading index 1
blk9_for_reading index 1
blk10_for_reading index 1
blk11_for_reading index 1

```
dispatch-barrier-sync 函数会等待追加到Concurrent Dispatch Queue上的并行执行的处理全部结束以后，再将指定的处理追加到该Concurrent Dispatch Queue中。然后在由Concurrent Dispatch Queue函数追加的处理执行完毕后。Concurrent Dispatch Queue才恢复为一般的动作，追加到该Concurrent Dispatch Queue的处理又开始并行执行。


#### 关于dispatch-suspend(挂起)/dispatch-resume(恢复)

**dispatch-suspend/dispatch-resume一般用于当追加大量处理到Dispatch Queue时，在追加处理的过程中有时希望不执行已追加的处理。**

```
    dispatch_queue_t queue = dispatch_queue_create("com.example.gcd.queue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_suspend(queue);
    dispatch_async(queue, ^{
        NSLog(@"blk0");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"blk1");
    });
    sleep(2);
    dispatch_resume(queue);
    // 两秒后恢复queue，然后才会执行queue里面的操作
```

#### 关于dispatch-sync

**dispatch-sync相对于dispatch-async的区别就在于它是就是同步的线程操作，只有指定的block完成以后dispatch-sync才会返回。**

但是dispatch-sync会带来一些死锁的情况。

`将主线程的Main Disaptch Queue，在主线程中执行dispatch_sync就会造成死锁。`

```
    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_sync(queue, ^{
        NSLog(@"lock");
    });
```

因为主线程正在执行上述代码块，所以此时的block无法执行到Main Disaptch Queue,由于Main Disaptch Queue是Serial Dispatch Queue；但是由于block无法执行，所以dispatch-sync就会一直等待block的执行，主线程此时死锁。

下面的例子也是同样的道理。

```
    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_async(queue, ^{
        dispatch_sync(queue, ^{
            NSLog(@"lock");
        });
    });
```

所以由此可知，我把Main Disaptch Queue替换成任何Serial Dispatch Queue都会造成死锁的问题。

```
    dispatch_queue_t queue = dispatch_queue_create("com.example.gcd.queue", NULL);
    dispatch_async(queue, ^{
        dispatch_sync(queue, ^{
            NSLog(@"lock");
        });
    });
```

####关于Dispatch Semaphore的问题

**Dispatch Semaphore主要用于信号量的控制并发，当处理一系列线程的时候，当数量达到一定量时，我们需要控制线程的并发量。**

在GCD中有三个函数是semaphore的操作，分别是：

* dispatch-semaphore-create　　　创建一个初始指定数量的信号量
* dispatch-semaphore-signal　　　发送一个信号，使信号量+1，计数为0时等待，计数为1或大于1时，减去1而不等待。
* dispatch-semaphore-wait　　　　等待信号，使信号量-1



```
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(5);
    dispatch_queue_t global = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    for (NSInteger index = 0; index < 50; index++) {
        /**
         *等待处理，直到信号量>0，信号量减1，当信号量为0时不需要在减1
         */
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        dispatch_async(global, ^{
            NSLog(@"thread %ld", index);
            /**
             *数据处理完毕，信号量加1
             */
            dispatch_semaphore_signal(semaphore);
        });
    }
```


### 总结

**GCD算是OC中一项很基础的知识了，灵活使用GCD会很大程度上提高我们的代码质量。**








