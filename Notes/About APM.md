#Unity3D 移动游戏  APM 监控笔记

近期公司有考虑到一个项目，为此进行了一系列调研，写下来当个笔记。考虑到游戏方面可能只关注于业务层面，而实际上的性能方面可能稍有欠缺，需要为此开发一个SDK用来检测游戏App的性能等等。想到哪写到哪吧。


先从一些基础的性能检测分析开始。

###CPU占用率的检测

线程是调度和分配的基本单位，所以只要获取应用所有线程占用CPU的情况即可得知CPU占用率

iOS是基于Apple Darwin内核，由Kernel，XNU，Runtime组成，XNU是Darwin的内核，意为X is not Unix，是混合内核。由Mach微内核和BSD组成.Mach为轻量级平台，只完成操作系统的基本职责比如进程和线程，虚拟内存管理，任务调度，进程通信和消息传递机制。其他工作入文件操作和设备访问由BSD完成.

iOS线程基于Mach线程技术

```
struct thread_basic_info {
	time_value_t    user_time;      /* user run time */
    time_value_t    system_time;    /* system run time */
    integer_t       cpu_usage;      /* scaled cpu usage percentage */
    policy_t        policy;         /* scheduling policy in effect */
    integer_t       run_state;      /* run state (see below) */
    integer_t       flags;          /* various flags (see below) */
    integer_t       suspend_count;  /* suspend count for thread */
    integer_t       sleep_time;     /* number of seconds that thread
                                           has been sleeping */
}
```

每一个BSD进程都对应着一个Mach Task
通过API task_threads 获取一个task下所有的线程列表。

```
kern_return_t task_threads 
(
	task_t target_task,
	thread_act_array_t *act_list,
	mach_msg_type_number_t *act_listCnt
);

/*  
	第一个参数为当前任务 
	第二个参数为保存该任务下的所有线程信息的数组 
    第三个参数为线程的个数。 
*/
```

获取到线程数组后，通过API  thread_info 来获取具体线程的信息

```
kern_return_t thread_info 
(
	thread_act_t target_act,
	thread_flavor_t flavor,
	thread_info_t thread_info_out,
	mach_msg_type_number_t *thread_info_outCnt
);

/*  
	第一个参数为当前任务
	第二个参数为查询指定的信息
	第三个参数为存储返回信息的缓存区
	第四个参数为该信息的长度(字节)
*/
```
当按照以上API调用后，我们便可以从thread_info_out中获取到想要查询的线程信息了，其中的cpu_usage即为该线程使用CPU情况，将任务下所有线程的情况想加即可得到CPU占用情况。

记得在方法的最后调用 vm_deallocate();防止内存泄漏
可以考虑为了更准确去掉监控线程的占用率。



###内存的监控

物理内存(RAM)同样是系统中稀缺的资源，特别容易产生竞争，所以应用内存与性能直接相关。iOS无交互空间作为备选资源，使用Jetsam来处理系统低RAM事件，通常做法就是Killer Out-Of-Memory。

监控App使用的内存，系统也给我们提供了好方法。mach_task_basic_info 结构体存储了Mach层上当前task的内存使用情况。resident_size为物理内存，virtual_size为虚拟内存。


```
#define MACH_TASK_BASIC_INFO  20
struct mach_task_basic_info {
	mach_vm_size_t virtual_size;
	mach_vm_size_t resident_size;
	mach_vm_size_t resident_size_max;
	time_value_t user_time;
	time_value_t system_time;
	policy_t policy;
	integer_t suspend_count;
};
```

和cpu类似  通过api

```
kern_return_t task_info
(
	task_t task;
	task_flavor_t flavor,
	task_info_t task_info_out,
	mach_msg_number_t *task_info_outCnt
);
```

可获取target_task信息。但注意一点，Xcode给我们提供的Debug Gauges中，显示的内存使用情况和resident_size并不相同。说明我们取到的斌故事真实的内存使用情况。
在WebKit源码中，MemoryFootprintCocoa.app中，

```
size_t memoryFootprint()
{
    task_vm_info_data_t vmInfo;
    mach_msg_type_number_t count = TASK_VM_INFO_COUNT;
    kern_return_t result = task_info(mach_task_self(), TASK_VM_INFO, (task_info_t) &vmInfo, &count);
    if (result != KERN_SUCCESS)
        return 0;
    return static_cast<size_t>(vmInfo.phys_footprint);
}
```
可以发现真实内存取的是phys_footprint属性。

###帧率(卡顿检测)

作为游戏应用，卡顿情况对于用户来说是最直接的糟糕体验。所以有必要进行卡顿的检测。

####先说说为什么会出现卡顿

说卡顿之前，我们要了解一下画面显示在屏幕上面的过程。iOS上完成图形的显示依靠的是CPU与GPU的协同工作。其中，CPU负责计算显示内容，包括视图创建、布局计算、图片解码、文本绘制等，完成这些工作计算后，CPU会将这些内容提交到GPU，GPU进行变换、合成、渲染，最后将渲染结果提交到FrameBuffer缓冲区，当垂直同步新号V-Sync 到来时，显示到屏幕上。

iOS的同步机制是什么，iOS采用的是双缓冲机制，即为有两个缓冲区，如果没有同步机制，那么当视频控制器还未将上一帧的内容显示完成时，GPU已经提供了新的一帧动画，那么缓冲区将会交换，视频控制器会将新的一帧的数据的下半部分显示到屏幕上，这样就会出现画面撕裂的情况，所以需要V-Sync 同步信号，只有当同步信号来临时，GPU才会去进行新一帧的渲染和对缓冲区的刷新。

了解完刷新机制，再来说说为什么会产生卡顿。即为当CPU和GPU完成一帧的任务的时间错过了下一次同步信号,那么此帧将不会被显示，屏幕显示的还是之前帧的画面，这就是卡顿的原因。


####如何监控卡顿

我们很简单能想到的方案有两种

- FPS监控：最容易想到的方案，帧率越高画面越流畅。
- 主线程的监控：业内常用方法，开辟一个子线程来监控主线程的RunLoop。例如美团的Hertz

这里我们考虑一下，由于FPS刷新频率特别快，抖动情况太大。所以我们很难通过比较FPS来判断卡顿情况。所以这里主要考虑通过主线程监控，当然FPS监控、CPU占用率都可以作为综合卡顿检测的指标范围。

当我们监控到卡顿情况时，需要找出卡顿的具体原因，所以需要在发生卡顿时保存应用上下文--堆栈调用和运行日志信息，如此便可以快速定位卡顿的问题来源。

具体实现思路：开辟一个子线程，实时计算kCFRunLoopBeforeSources和kCFRunLoopAfterWaiting两个状态区域之间的耗时，制定一个卡顿判断的阈值后，来判定该耗时是否发生卡顿情况。堆栈信息的收集暂时考虑的方法为，自定义NSException，使用NSSetUncaughtExceptionHandler来获取堆栈信息。更好的还有一些三方库比如PLCrashReport等。

```
// 监控卡顿

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    MyClass *object = (__bridge MyClass*)info;
    
    // 记录状态值
    object->activity = activity;
    
    // 发送信号
    dispatch_semaphore_t semaphore = moniotr->semaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)registerObserver
{
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                            kCFRunLoopAllActivities,
                                                            YES,
                                                            0,
                                                            &runLoopObserverCallBack,
                                                            &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    
    // 创建信号
    semaphore = dispatch_semaphore_create(0);
    
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            // 假定连续5次超时50ms认为卡顿(当然也包含了单次超时250ms)
            long st = dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            if (st != 0)
            {
                if (activity==kCFRunLoopBeforeSources || activity==kCFRunLoopAfterWaiting)
                {
                    if (++timeoutCount < 5)
                        continue;
                    // 检测到卡顿，进行卡顿上报
                }
            }
            timeoutCount = 0;
        }
    });
}          


```

###电量的监控

电量比较简单，苹果也给我们提供了简单的方法。
首先考虑UIDevice，这是设备信息。可以从中获取电池相关信息。

[UIDevice currentDevice] 即为当前设备。要获取电池信息需要把属性 batteryMonitorEnabled 置为YES
但是UIDeivce的batterLevel虽然可以显示电量，但是由于其粗粒度，并不十分精确，生产环境不建议使用。

此时便引出了另一个方案，苹果有一个私有库IOKit.framework，它可以被用来获取硬件设备的详细信息，这里获取的电池信息十分精确。(具体代码此处不列，可以使用动态链接的方式)



以上便是简单的基础性能的监控笔记。关于网络方面，比较复杂，后续会进行补充。



##Unity3D相关

回到正题，以上基础的性能检测我们可以在native层做完，比较简单，现在需要考虑到的是异常的检测和Unity3D相关的一些东西。关于Unity3D跨平台生成iOS游戏的项目，我们可以分为两层来看。在Unity3D引擎中我们使用的是C#脚本，即为C#层面，跨平台出包后会生成一套native语言的包，即为native层。


###C#层面

对于C#的异常捕获，有以下方法

- 使用Unity引擎中提供的相关类 'CrashReport'，当应用程序崩溃时，Unity会尝试收集有用信息,比如代码位置和线程堆栈跟踪。在下一次启动程序时，则可以通过此API来访问所有崩溃信息。(这也是Unity项目中Debugging and crash reporting的具体表现)
- 通过注册Application.logMessageReceeivedThreaded日志回调事件 (Unity 4.0之前版本，请使用 'Application.RegisterLogCallback')，获取堆栈信息和日志消息。异常会触发该事件，所以可以使用(release模式下也可以！！！)
- 通过注册AppDomain.CurrentDomain.UnhandledException回调，不过由于Unity的函数基本都有try catch，所以这块也基本不会出现未捕获的异常。

可以看看腾讯Bugly的注册异常源码

```
private static void _RegisterExceptionHanlder () {
	try{
		// hold any one instance
		#if UNITY_5
		Application.logMessageReceived += _OnLogCallbackHandler;
		#else
		Application.RegisterLogCalback (_OnLogCallbackHandler);
		#endif
		AppDomain.CurrentDomain.UnhandledException += _OnUncaughtExceptionHandler;
		_isInitialized = true;
		Debug (null, "Register the log callback in Unity {0}", Application.unityVersion);
	} catch {}
	SetUnityVersion ();
}
```

原生层面收集C#异常代码并没有什么好方法，C#脚本语言转换成C++后，获取不到C#符号表，无法还原异常信息。


###Native(iOS)层面

如前文，通过NSSetUncaughtExceptionHandler来监听异常发生，在回调中拿到exception的信息以及堆栈信息。(Unity项目中 Debugging and crash reporting 默认实现)。此层无太大意义，异常堆栈信息大部分是Unity Runtime和系统库方法,无C#符号表情况下，对业务无太大帮助。



了解完全部的调研部分后，我们考虑如何集成。
目前想到两种方案

方式一： 游戏开发人员在项目中集成该SDK。(业内主流第三方平台实现方案， 比如 Bugly  Fabric)

#####需要开发一款SDK，包括：
- C#插件，负责注册相关回调函数，用于收集异常数据等，其中提供原生SDK的接口封装
- 原生SDK,负责与C#插件进行通信，性能数据的手机。汇总后最后上传到服务器。


方式二： 游戏开发人员仅仅提供一份release安装包，拿到该安装包后，通过注入方式进行性能数据收集。
####(条件：安装包不做BundleID，越狱等安全检测，因为需要逆向)
####但是该方案下，动态库没有C#环境，所以无法注册C#回调函数，所以无法收集C#异常信息。
- 编写动态库
- 通过MonkeyDev等方式，逆向注入动态库，并进行重签名。



