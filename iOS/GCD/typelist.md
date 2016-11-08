# GCD 知识点总结

## 介绍

GCD 属于系统级别的线程管理，在 Dispatch queue 中执行需要执行的任务性能非常的高。

GCD 中的 FIFO 队列称为 dispatch queue，用来保证先进来的任务先得到执行.(猜测为 first in frist out)

一共有主队列，三个优先级不同的后台队列，以及优先级更低的 `Background` 队列

* Main Queue
* Hight Priority Queue
* Default Priority Queue
* Low Priority Queue
* Background Priority Queue（用于I/O）优先级最低

可以自行创建队列，一般放在 `Default Priority Queue` 和 `Main Queue`

操作是在多线程上还是单线程主要是看队列的类型和执行方法

* 并行队列异步执行才能在多线程
* 并行队列同步执行就只会在主线程执行
  
## 基本
 
1. 系统已经存在的两个队列

	* 全局队列，一个并行的队列
	
	```oc
	dispatch_get_global_queue
	```	
	
	* 主队列，主线程中的唯一队列，一个串行队列
	
	```oc
	dispatch_get_main_queue
	```

2. 自定义队列
	
	* 串行队列

	```oc
	dispatch_queue_create("com.heronlyj.serialqueue", DISPATCH_QUEUE_SERIAL)
	```
	
	* 并行队列
	
	```oc
	dispatch_queue_create("com.heronlyj.concurrentqueue", DISPATCH_QUEUE_CONCURRENT)
	```

3. 同步异步
	
	* 同步
	
	```oc
	dispatch_sync(..., ^(block))
	```
	
	* 异步
	
	```oc
	dispatch_async(..., ^(block))
	```
	
## 队列

#### 队列类型

* 主队列 Main dispatch queue: 全局 serial 队列
* 串行 Serial (private dispatch queues): 同时只执行一个任务，多个 serial 依然为并发执行
* 全局并行队列 Concurrent (global dispatch queue): 可以并发的执行多个任务，但执行完成顺序是随机的，用户只能获取不能创建

	```
	dipatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH,0);
	```
* user create 自定义队列
		
#### 自定义队列

队列类型可以一下三种选项

* NULL  *(如果设置为 NULL 就默认是 串行 DISPATCH_QUEUE_SERIAL)*
* DISPATCH_QUEUE_SERIAL 串行
* DISPATCH_QUEUE_CONCURRENT 并行

```oc
dispatch_queue_t queue = dispatch_queue_create("队列名", 队列类型);
```
设置优先级的两种方式

* dipatch_queue_attr_make_with_qos_class

```oc
dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, -1);
dispatch_queue_t queue = dispatch_queue_create("队列名", attr);
```

* dispatch_set_target_queue

```oc
//需要设置优先级的queue
dispatch_queue_t queue = dispatch_queue_create("队列名",NULL); 
//参考优先级
dispatch_queue_t referQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0); 
//设置queue和referQueue的优先级一样
dispatch_set_target_queue(queue, referQueue); 

////////////////////////////////////////////////////////////////////////////////////////////

// 让多个串行和并行队列在统一一个串行队列里串行执行
dispatch_queue_t serialQueue = dispatch_queue_create("serialqueue", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t firstQueue = dispatch_queue_create("firstqueue", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t secondQueue = dispatch_queue_create("secondqueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_set_target_queue(firstQueue, serialQueue);
dispatch_set_target_queue(secondQueue, serialQueue);

dispatch_async(firstQueue, ^{
    NSLog(@"1");
    [NSThread sleepForTimeInterval:3.f];
});
dispatch_async(secondQueue, ^{
    NSLog(@"2");
    [NSThread sleepForTimeInterval:2.f];
});
dispatch_async(secondQueue, ^{
    NSLog(@"3");
    [NSThread sleepForTimeInterval:1.f];
});

// 按顺序执行的 1，2，3

```

#### 优先级参数 

* Quality of Service(QoS，iOS8)枚举, 推荐使用这种参数

值 | 说明
----- | ------
QOS_CLASS_USER_INTERACTIVE | 最高优先级，交互级别。使用这个优先级会占用几乎所有的<br/>系统CUP和I/O带宽，仅限用于交互的UI操作，比如处理点<br/>击事件，绘制图像到屏幕上，动画等
QOS_CLASS_USER_INITIATED | 次高优先级，用于执行类似初始化等需要立即返回的事件
QOS_CLASS_DEFAULT | 默认优先级，当没有设置优先级的时候，线程默认优先级。 一般情况下用的都是这个优先级
QOS_CLASS_UTILITY | 普通优先级，主要用于不需要立即返回的任务
QOS_CLASS_BACKGROUND | 后台优先级，用于用户几乎不感知的任务
QOS_CLASS_UNSPECIFIED | 未知优先级，表示服务质量信息缺失

* dispatch_queue_priority_t

参数 | 值 | 对应的 qos
----- | ------ | ------
DISPATCH_QUEUE_PRIORITY_HIGH | 2 | QOS_CLASS_USER_INITIATED
DISPATCH_QUEUE_PRIORITY_DEFAULT | 0 | QOS_CLASS_DEFAULT
DISPATCH_QUEUE_PRIORITY_LOW | -2 | QOS_CLASS_UTILITY
DISPATCH_QUEUE_PRIORITY_BACKGROUND | INT16_MIN | QOS_CLASS_BACKGROUND


#### 使用

方法 | 效果
----- | ------
dispatch_once |  单例的写法
dispatch_async | 异步执行
dispatch_sync | 同步执行
dispatch_after | 延后执行 只是延后提交 block，不是延时立即执行
dispatch_barrier_async | 解决多线程并发读写同一个资源发生死锁
dispatch_apply | 进行快速迭代, 会阻塞主线程
Dispatch_groups | Block 组合
dispatch_group_notify | dispatch_group_notify 和 dispatch_group_wait 的区别就是是异步执行闭包的, 当 dispatch groups 中没有剩余的任务时闭包才执行。这里是指明在主队列中执行
dispatch_group_enter | dispatch_group_enter 是通知 dispatch group 任务开始了， dispatch_group_enter 和 dispatch_group_leave 是成对调用，不然程序就崩溃了
dispatch_group_leave | 保持和 dispatch_group_enter 配对, 通知任务已经完成
dispatch_group_wait | 等待所有任务都完成直到超时。如果任务完成前就超时了，函数会返回一个<br/>非零值，可以通过返回值判断是否超时。也可以用 DISPATCH_TIME_FOREVER 表示一直等
dispatch_block_cancel | iOS8 后 GCD 支持对 dispatch block 的取消


----
参考
[细说GCD（Grand Central Dispatch）如何用](https://github.com/ming1016/study/wiki/%E7%BB%86%E8%AF%B4GCD%EF%BC%88Grand-Central-Dispatch%EF%BC%89%E5%A6%82%E4%BD%95%E7%94%A8)




