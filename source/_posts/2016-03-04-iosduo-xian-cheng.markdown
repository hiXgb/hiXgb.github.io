---
layout: post
title: "iOS多线程"
date: 2016-03-04 15:36:48 +0800
comments: true
categories: 

---

#iOS多线程
-------------------
### NSThread
	
基于线程

### GCD

基于task

- 同步异步

	- dispatch_sync在当前线程同步执行，当前线程被阻塞，后续的代码片段需要等sync任务执行完返回后才被执行，即使加入到并行队列也不开新线程。
	- dispatch_async不阻塞当前线程，在当前线程直接返回，后续代码片段接着执行，会开新线程。

- 串行并行队列

	```
	dispatch_queue_t申明时要注意是否在ARC环境下使用，可用如下方式使用
	
	#if OS_OBJECT_USE_OBJC
@property (nonatomic, strong) dispatch_queue_t queue;
#else
@property (nonatomic, assign) dispatch_queue_t queue;
#endif
```
	创建方式如下：
	```
	dispatch_queue_t concurrentQueue = dispatch_queue_create("com.company.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
    ```
    ```
    dispatch_queue_t serialQueue = dispatch_queue_create("com.company.serialQueue", DISPATCH_QUEUE_SERIAL);
    ```

	- 串行队列一次只能执行一个任务，根据入队的顺序执行任务,一个任务执行完下一个任务开始的时间是不确定的	
	- 并行队列可以同时执行多个任务，系统会维护一个线程池来保证并行队列的执行。系统只保证根据入队的顺序开始执行，因此task的开始执行时间是顺序的，但是每个task的时长不确定，有可能会出现多个task在同一时间需要同时执行，此时就需要另开线程来执行了。线程池会根据当前任务量自行安排线程的数量，以确保任务尽快执行。（因为是系统维护，有可能出现某个时间段线程数过多的问题）
	
	
- 使用GCD常见死锁 

	在同一个串行队列中又用dispatch_sync的方式添加了任务
	
	```
	//serialQueue是同一个串行队列
	dispatch_async(serialQueue, ^{
    //some tasks

    dispatch_sync(serialQueue, ^{  //发生死锁

    });
});
	```
	在主线程dispatch_sync就是上述的常见场景
	
	```
	//在主线程执行
	dispatch_sync(mainQueue, ^{  //发生死锁

    });
	```
	
- 部分api记录

	- dispatch_apply，在指定的queue循环执行task
	- dispatch_barrier，在并行队列中，有的时候我们需要让某个任务单独执行，也就是他执行的时候不允许其他任务执行。这时候dispatch_barrier就派上了用场。
	
	使用dispatch_barrier将任务加入到并行队列之后，任务会在前面任务全部执行完成之后执行，任务执行过程中，其他任务无法执行，直到barrier任务执行完成。
	一个常见的场景是属性的读写，例子如下：
	
	```
	- (void)setName:(NSString *)name
{
    //写的时候阻塞住，等写操作结束后继续后续操作
    dispatch_barrier_async(_concurrentQueue, ^{
        _name = name;
    });
}
- (NSString *)name
{
    __block NSString *localString;
    dispatch_sync(_concurrentQueue, ^{
        localString = _name;
    });
    return localString;
}
```

- 和NSOpration的区别

###NSOpration

基于队列