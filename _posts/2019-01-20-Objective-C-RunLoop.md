---
layout:     post
title:      Objective-C-RunLoop
date:       2019-01-20
---

# RunLoop简介

顾名思义：

- 运行循环
- 在程序运行过程中循环做一些事情

应用范畴：

- 定时器（Timer）、PerformSelector
- GCD Async Main Queue
- 事件响应、手势识别、界面刷新
- 网络请求
- AutoreleasePool

如果没有RunLoop，执行完第13行代码后，会即将退出程序

```c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        NSLog(@"Hello, World!");  // line：13
    }
    return 0;
}
```

如果有RunLoop：

- 程序并不会马上退出，而是保持运行状态
- RunLoop的基本作用
  - 保持程序的持续运行
  - 处理App中的各种事件（比如触摸事件、定时器事件等）
  - 节省CPU资源，提高程序性能：该做事时做事，该休息时休息
  - ......

```objective-c
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, 
                                 NSStringFromClass([AppDelegate class]));
    }
}

// 伪代码如下
int main(int argc, char * argv[]) {
    @autoreleasepool {
        int retval = 0;
        do {
            // 睡眠中等待消息
            int message = sleep_and_wait();
            // 处理消息
            retval = process_message(message);
        } while (0 == retval);
        return 0;
    }
}
```

# RunLoop对象

iOS中有2套API来访问和使用RunLoop：

- Foundation：NSRunLoop
- Core Foundation：CFRunLoopRef

NSRunLoop和CFRunLoopRef都代表着RunLoop对象，NSRunLoop是基于CFRunLoopRef的一层OC包装，CFRunLoopRef是开源的，https://opensource.apple.com/tarballs/CF/

```objective-c
NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
CFRunLoopRef runLoop2 = CFRunLoopGetCurrent();
```

# RunLoop与线程

- 每条线程都有唯一的一个与之对应的RunLoop对象
- RunLoop保存在一个全局的Dictionary里，线程作为key，RunLoop作为value
- 线程刚创建时并没有RunLoop对象，RunLoop会在第一次获取它时创建，即`[NSRunLoop currentRunLoop]`

- RunLoop会在线程结束时销毁
- 主线程的RunLoop已经自动获取（创建），子线程默认没有开启RunLoop

# RunLoop相关的类

Core Foundation中关于RunLoop的5个类：

- CFRunLoopRef
- CFRunLoopModeRef
- CFRunLoopSourceRef
- CFRunLoopTimerRef
- CFRunLoopObserverRef

CFRunLoopRef结构为：

```c
typedef struct __CFRunLoop * CFRunLoopRef;

struct __CFRunLoop {
    pthread_t _pthread;
	CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;  // 集合里边为 CFRunLoopModeRef 类型
    // 主要信息为以上5个
    
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    uint32_t _winthread;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```

CFRunLoopModeRef结构为：

```c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFStringRef _name;
	CFMutableSetRef _sources0;  // 集合里边为 CFRunLoopSourceRef 类型
    CFMutableSetRef _sources1;  // 集合里边为 CFRunLoopSourceRef 类型
    CFMutableArrayRef _observers; // 集合里边为 CFRunLoopTimerRef 类型
    CFMutableArrayRef _timers;	  // 集合里边为 CFRunLoopObserverRef 类型
	//以上5个为主要信息
    
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    Boolean _stopped;
    char _padding[3];
    
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

![avatars](https://ws2.sinaimg.cn/large/006tNc79ly1fzca0uww92j30l00dimz3.jpg)


- Souce0
  - 触摸事件处理
  - performSelector:onThread:

- Source1
  - 基于Port的线程间通信
  - 系统事件捕捉（屏幕点击，Source1先响应，然后传给Souce0处理）

- Timers
  - NSTimer
  - performSelector:withObject:afterDelay:

- Observers
  - 用于监听RunLoop的状态
  - UI刷新（BeforeWaiting）（ `self.view.backgroundColor = [UIColor redColor];`不是立即执行，而是在睡觉之前执行的）
  - Autorelease pool（BeforeWaiting）
  
## CFRunLoopModeRef

- CFRunLoopModeRef代表RunLoop的运行模式
- 一个RunLoop包含若干个Mode，每个Mode又包含若干个Source0/Source1/Timer/Observer
- RunLoop启动时只能选择其中一个Mode，作为currentMode
- 如果需要切换Mode，只能退出当前Loop，再重新选择一个Mode进入
  - 不同组的Source0/Source1/Timer/Observer能分隔开来，互不影响

- 如果Mode里没有任何Source0/Source1/Timer/Observer，RunLoop会立马退出

常见的2种Mode：

1. kCFRunLoopDefaultMode（NSDefaultRunLoopMode）：App的默认Mode，通常主线程是在这个Mode下运行
2. UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
3. NSRunLoopCommonModes 包括 NSDefaultRunLoopMode，UITrackingRunLoopMode

## CFRunLoopObserverRef

RunLoop的状态：

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),			// 即将进入Loop
    kCFRunLoopBeforeTimers = (1UL << 1),	// 即将进入Timer
    kCFRunLoopBeforeSources = (1UL << 2),	// 即将处理Source
    kCFRunLoopBeforeWaiting = (1UL << 5),	// 即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),	// 刚从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),			// 即将退出Loop
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

# RunLoop的运行逻辑

![avatars](https://ws1.sinaimg.cn/large/006tNc79ly1fzcr31j1bqj317o0l40vz.jpg)

1. 通知Observers：进入Loop
2. 通知Observers：即将处理Timers
3. 通知Observers：即将处理Sources
4. 处理Blocks
5. 处理Source0（可能会再次处理Blocks）
6. 如果存在Source1，就跳转到第8步
7. 通知Observers：开始休眠（等待消息唤醒）
8. 通知Observers：结束休眠（被某个消息唤醒）
   - 处理Timer
   - 处理GCD Async To Main Queue
   - 处理Source1

9. 处理Blocks
10. 根据前面的执行结果，决定如何操作
    - 回到第02步
    - 退出Loop

11. 通知Observers：退出Loop

### RunLoop休眠的实现原理

![avatars](https://ws4.sinaimg.cn/large/006tNc79ly1fzctjv29qhj30xa0i2aai.jpg)

RunLoop的休眠是切换到内核态，真正让线程休眠。

# RunLoop在实际开发中的应用

- 控制线程生命周期（线程保活）

- 解决NSTimer在滑动时停止工作的问题

- 监控应用卡顿

- 性能优化

## 解决NSTimer在滑动时停止工作的问题

```objective-c
    static int count = 0;
    NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * 	   _Nonnull timer) {
        NSLog(@"%d", ++count);
    }];
//    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
//    [[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
    
    // NSDefaultRunLoopMode、UITrackingRunLoopMode才是真正存在的模式
    // NSRunLoopCommonModes并不是一个真的模式，它只是一个标记
    // timer能在_commonModes数组中存放的模式下工作  CFMutableSetRef _commonModes; 
	// 它里边包含NSDefaultRunLoopMode、UITrackingRunLoopMode这两种模式
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

## 子线程保活

```objective-c
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

typedef void (^HPermenantThreadTask)(void);

@interface HPermenantThread : NSObject

/**
 开启线程
 */
- (void)run;

/**
 在当前子线程执行一个任务
 */
- (void)executeTask:(HPermenantThreadTask)task;

/**
 结束线程
 */
- (void)stop;

@end

NS_ASSUME_NONNULL_END
    
#import "HPermenantThread.h"

/** MJThread **/
@interface HThread : NSThread
@end
@implementation HThread
- (void)dealloc {
    NSLog(@"%s", __func__);
}
@end

/** HPermenantThread **/
@interface HPermenantThread()
@property (strong, nonatomic) HThread *innerThread;
@property (assign, nonatomic, getter=isStopped) BOOL stopped;
@end

@implementation HPermenantThread

#pragma mark - public methods

- (instancetype)init {
    if (self = [super init]) {
        self.stopped = NO;

        __weak typeof(self) weakSelf = self;

        self.innerThread = [[HThread alloc] initWithBlock:^{
            [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];

            while (weakSelf && !weakSelf.isStopped) {
                [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
            }
        }];
    }
    return self;
}

- (void)run {
    if (!self.innerThread) return;

    [self.innerThread start];
}

- (void)executeTask:(HPermenantThreadTask)task {
    if (!self.innerThread || !task) return;

    [self performSelector:@selector(__executeTask:) onThread:self.innerThread withObject:task waitUntilDone:NO];
}

- (void)stop {
    if (!self.innerThread) return;

    [self performSelector:@selector(__stop) onThread:self.innerThread withObject:nil waitUntilDone:YES];
}

- (void)dealloc {
    NSLog(@"%s", __func__);

    [self stop];
}

#pragma mark - private methods
- (void)__stop {
    self.stopped = YES;
    CFRunLoopStop(CFRunLoopGetCurrent());
    self.innerThread = nil;
}

- (void)__executeTask:(HPermenantThreadTask)task {
    task();
}

@end    
    
```