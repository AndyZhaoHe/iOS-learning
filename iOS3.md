# iOS RunLoop 题库详解

> 来源：根据用户上传的 `03-RunLoop.pptx` 整理。

---

# 一、RunLoop 基础概念

## 1. 什么是 RunLoop？

**答案：**

RunLoop 直译就是“运行循环”。它本质上是一个循环机制，让线程在有任务时执行任务，没有任务时进入休眠状态，而不是直接退出。

在 iOS 中，RunLoop 的主要作用有：

1. 保持程序持续运行。
2. 处理 App 中的各种事件，比如触摸事件、Timer 事件、手势事件、界面刷新等。
3. 节省 CPU 资源，提高性能。
4. 管理线程生命周期。
5. 支持 AutoreleasePool 的释放时机。
6. 支持 GCD 主队列任务处理。
7. 支持 performSelector 延迟执行。
8. 支持网络请求、端口通信等。

可以简单理解为：

```objc
do {
    处理事件;
    如果没有事件，线程休眠;
    有事件时，线程被唤醒继续处理;
} while (没有退出);
```

**追问：如果没有 RunLoop 会怎么样？**

如果没有 RunLoop，程序执行完 main 函数中的代码后就会直接退出。  
iOS App 之所以能一直运行，是因为主线程开启了 RunLoop，它不断等待和处理事件。

---

## 2. RunLoop 的核心作用是什么？

**答案：**

RunLoop 的核心作用可以总结为三点：

### 1. 保持线程不退出

比如主线程启动后不会执行完就结束，而是一直等待用户事件、Timer、网络回调等。

### 2. 处理各种事件

包括：

```text
触摸事件
手势识别
Timer 定时器
PerformSelector
GCD Async Main Queue
界面刷新
AutoreleasePool
网络事件
端口通信
```

### 3. 节省 CPU

RunLoop 并不是一直死循环占用 CPU。

它的机制是：

```text
有任务：处理任务
没任务：进入休眠
有新消息：被唤醒
```

底层通过 `mach_msg()` 实现线程休眠和唤醒。

---

## 3. RunLoop 是死循环吗？

**答案：**

RunLoop 是一种循环结构，但不是普通意义上的死循环。

普通死循环：

```objc
while (1) {
    // 一直执行，CPU 占用很高
}
```

RunLoop：

```objc
while (没有退出) {
    有事件就处理;
    没事件就休眠;
}
```

区别是 RunLoop 在没有任务时会让线程休眠，不会一直占用 CPU。

所以 RunLoop 是“事件驱动的循环”，不是简单的 CPU 空转死循环。

---

# 二、RunLoop 对象与 API

## 4. iOS 中有几套 API 可以访问 RunLoop？

**答案：**

iOS 中有两套 API 可以访问 RunLoop：

### 1. Foundation 层

```objc
NSRunLoop
```

常用方法：

```objc
[NSRunLoop currentRunLoop];
[NSRunLoop mainRunLoop];
```

### 2. Core Foundation 层

```objc
CFRunLoopRef
```

常用函数：

```objc
CFRunLoopGetCurrent();
CFRunLoopGetMain();
```

### 二者关系

```text
NSRunLoop 是对 CFRunLoopRef 的 Objective-C 封装
CFRunLoopRef 是 Core Foundation 层的底层实现
```

也就是说，`NSRunLoop` 更偏上层，使用方便；`CFRunLoopRef` 更底层，功能更强。

---

## 5. NSRunLoop 和 CFRunLoopRef 有什么区别？

**答案：**

| 对比项 | NSRunLoop | CFRunLoopRef |
|---|---|---|
| 所属框架 | Foundation | Core Foundation |
| 语言风格 | Objective-C 对象 | C 结构 / 引用 |
| 使用方式 | 面向对象 | C 函数 |
| 底层能力 | 封装较多 | 更底层、更灵活 |
| 关系 | 包装层 | 真正底层实现 |

### 总结

`NSRunLoop` 是对 `CFRunLoopRef` 的封装。  
真正底层运行逻辑主要由 `CFRunLoopRef` 实现。

---

# 三、RunLoop 和线程

## 6. RunLoop 和线程是什么关系？

**答案：**

RunLoop 和线程是一一对应的关系。

核心规则：

```text
每一条线程都有唯一对应的 RunLoop
RunLoop 保存在全局 Dictionary 中
线程作为 key
RunLoop 作为 value
```

具体特点：

1. 主线程的 RunLoop 默认已经创建并启动。
2. 子线程默认不会自动开启 RunLoop。
3. 子线程的 RunLoop 在第一次获取时才会创建。
4. 线程结束时，对应的 RunLoop 也会销毁。
5. 每个线程只能有一个 RunLoop。

### 获取当前线程 RunLoop

```objc
NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
```

或者：

```objc
CFRunLoopRef runLoop = CFRunLoopGetCurrent();
```

---

## 7. 子线程默认有 RunLoop 吗？

**答案：**

子线程默认没有主动运行 RunLoop。

准确说：

```text
子线程有对应 RunLoop 的能力
但默认不会自动创建并运行
只有第一次获取 currentRunLoop 时才会创建
```

例如：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
});
```

这时候子线程的 RunLoop 会被创建。

但是，仅仅创建 RunLoop 不代表它会一直运行。  
如果想让子线程保持不退出，需要手动启动 RunLoop。

---

## 8. 如何让子线程常驻？

**答案：**

可以通过 RunLoop 让子线程保活。

示例：

```objc
@interface RZThread : NSThread
@end

@implementation RZThread
- (void)dealloc {
    NSLog(@"线程释放");
}
@end

@interface ViewController ()
@property (nonatomic, strong) RZThread *thread;
@property (nonatomic, assign) BOOL stopped;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    self.stopped = NO;

    self.thread = [[RZThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    [self.thread start];
}

- (void)run {
    NSLog(@"子线程启动：%@", [NSThread currentThread]);

    @autoreleasepool {
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];

        // 必须添加 Source / Timer / Observer 中至少一种
        [runLoop addPort:[NSPort port] forMode:NSDefaultRunLoopMode];

        while (!self.stopped) {
            [runLoop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
    }

    NSLog(@"子线程结束");
}

- (void)executeTask {
    if (!self.thread) return;

    [self performSelector:@selector(task)
                 onThread:self.thread
               withObject:nil
            waitUntilDone:NO];
}

- (void)task {
    NSLog(@"执行任务：%@", [NSThread currentThread]);
}

- (void)stopThread {
    if (!self.thread) return;

    [self performSelector:@selector(stop)
                 onThread:self.thread
               withObject:nil
            waitUntilDone:YES];
}

- (void)stop {
    self.stopped = YES;

    CFRunLoopStop(CFRunLoopGetCurrent());

    self.thread = nil;
}

- (void)dealloc {
    [self stopThread];
}

@end
```

### 关键点

RunLoop 如果没有任何 Source、Timer、Observer，会立刻退出。  
所以要添加一个 Port：

```objc
[runLoop addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
```

这样 RunLoop 才不会立即退出。

---

# 四、RunLoop 相关类

## 9. RunLoop 相关的核心类有哪些？

**答案：**

Core Foundation 中与 RunLoop 相关的核心类有 5 个：

```objc
CFRunLoopRef
CFRunLoopModeRef
CFRunLoopSourceRef
CFRunLoopTimerRef
CFRunLoopObserverRef
```

它们的关系是：

```text
RunLoop
 ├── Mode
 │    ├── Source0
 │    ├── Source1
 │    ├── Timer
 │    └── Observer
 ├── Mode
 │    ├── Source0
 │    ├── Source1
 │    ├── Timer
 │    └── Observer
```

一个 RunLoop 可以包含多个 Mode。  
每个 Mode 里面又可以包含多个 Source、Timer、Observer。

---

## 10. CFRunLoopModeRef 是什么？

**答案：**

`CFRunLoopModeRef` 表示 RunLoop 的运行模式。

一个 RunLoop 可以有多个 Mode，但是 RunLoop 每次运行时只能选择其中一个 Mode 作为当前模式。

核心规则：

```text
一个 RunLoop 包含多个 Mode
一个 Mode 包含 Source、Timer、Observer
RunLoop 每次只能运行在一个 Mode 下
切换 Mode 需要先退出当前 Loop，再进入另一个 Mode
不同 Mode 下的事件互不影响
如果当前 Mode 没有任何 Source、Timer、Observer，RunLoop 会立刻退出
```

---

## 11. 常见的 RunLoop Mode 有哪些？

**答案：**

常见 Mode 有：

### 1. NSDefaultRunLoopMode

也叫：

```objc
kCFRunLoopDefaultMode
```

这是 App 默认运行模式。  
主线程平时大多数情况下都运行在这个 Mode 下。

例如普通 Timer 默认就添加到这个 Mode。

### 2. UITrackingRunLoopMode

这是界面追踪模式。

当用户滑动 `UIScrollView`、`UITableView`、`UICollectionView` 时，主线程 RunLoop 会切换到这个 Mode。

它的作用是：

```text
保证滑动操作优先处理
避免被其他 Mode 中的事件干扰
让界面滑动更流畅
```

### 3. NSRunLoopCommonModes

注意：它不是一个真正的 Mode，而是一个标记集合。

常见包含：

```objc
NSDefaultRunLoopMode
UITrackingRunLoopMode
```

当把 Timer 添加到 `NSRunLoopCommonModes` 时，本质上是让 Timer 同时加入 Common Modes 里面包含的多个 Mode。

---

# 五、Mode 的作用

## 12. RunLoop 的 Mode 有什么作用？

**答案：**

Mode 的作用是隔离不同类型的事件。

例如：

```text
默认状态：NSDefaultRunLoopMode
滑动状态：UITrackingRunLoopMode
```

如果一个 Timer 只添加到了 `NSDefaultRunLoopMode`，当用户滑动 TableView 时，RunLoop 会切换到 `UITrackingRunLoopMode`，这时默认模式下的 Timer 就不会被处理。

所以 Mode 的作用是：

1. 隔离不同事件源。
2. 保证某些场景下的优先级。
3. 避免事件互相干扰。
4. 提高界面响应流畅度。

---

## 13. 为什么 TableView 滑动时 NSTimer 会暂停？

**答案：**

因为 `NSTimer` 默认添加在 `NSDefaultRunLoopMode` 中。

当用户滑动 `UITableView` 时，主线程 RunLoop 会切换到：

```objc
UITrackingRunLoopMode
```

此时 RunLoop 只会处理 `UITrackingRunLoopMode` 下的 Source、Timer、Observer。

如果 Timer 只在 `NSDefaultRunLoopMode` 中，它就不会被触发。

所以滑动时 Timer 看起来像“暂停”了。

---

## 14. 如何解决滑动 TableView 时 NSTimer 不响应的问题？

**答案：**

把 Timer 添加到 `NSRunLoopCommonModes`。

示例：

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:3.0
                                         target:self
                                       selector:@selector(timerAction)
                                       userInfo:nil
                                        repeats:YES];

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

或者：

```objc
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:3.0
                                                  target:self
                                                selector:@selector(timerAction)
                                                userInfo:nil
                                                 repeats:YES];

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

不过要注意，`scheduledTimerWithTimeInterval` 会默认把 Timer 添加到 `NSDefaultRunLoopMode`。  
如果再添加到 `NSRunLoopCommonModes`，需要理解它会被加入 Common Modes 对应的模式中。

更推荐写法：

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:3.0
                                         target:self
                                       selector:@selector(timerAction)
                                       userInfo:nil
                                        repeats:YES];

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

### Swift 写法

```swift
let timer = Timer(timeInterval: 3.0, repeats: true) { _ in
    print("timer fired")
}

RunLoop.main.add(timer, forMode: .common)
```

---

# 六、Source0 和 Source1

## 15. Source0 和 Source1 有什么区别？

**答案：**

RunLoop 的 Source 分为两类：

```objc
Source0
Source1
```

### Source0

Source0 是非基于 Port 的事件源。

常见场景：

```text
触摸事件处理
performSelector:onThread:
```

Source0 需要手动标记为待处理，然后唤醒 RunLoop。

例如用户点击屏幕后，系统事件最终会包装成 Source0 交给 App 处理。

### Source1

Source1 是基于 Port 的事件源，和内核事件、线程间通信有关。

常见场景：

```text
基于 Port 的线程间通信
系统事件捕捉
mach_msg
```

Source1 可以主动唤醒 RunLoop。

---

## 16. 触摸事件属于 Source0 还是 Source1？

**答案：**

触摸事件的底层接收和最终处理涉及两类 Source。

可以这样理解：

1. 系统先通过 Source1 接收硬件事件。
2. 然后封装成 App 可处理的事件。
3. 最终通过 Source0 分发给应用处理。

更常见的说法是：

```text
触摸事件处理属于 Source0
系统事件捕捉属于 Source1
```

所以回答时可以说：

> 用户触摸屏幕后，系统通过 Source1 接收事件，然后唤醒 RunLoop，事件经过处理后分发到应用层，最终通过 Source0 执行触摸事件回调。

---

# 七、Timer 和 RunLoop

## 17. Timer 和 RunLoop 是什么关系？

**答案：**

`NSTimer` 依赖 RunLoop 才能工作。

本质上：

```text
NSTimer 是对 CFRunLoopTimerRef 的封装
Timer 必须被添加到 RunLoop 的某个 Mode 中
只有 RunLoop 运行在对应 Mode 下，Timer 才会触发
```

例如：

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:1.0
                                         target:self
                                       selector:@selector(test)
                                       userInfo:nil
                                        repeats:YES];

[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
```

如果当前 RunLoop 没有运行，Timer 不会触发。  
如果 RunLoop 运行的 Mode 和 Timer 所在 Mode 不一致，Timer 也不会触发。

---

## 18. NSTimer 准确吗？

**答案：**

NSTimer 不是绝对准确的。

原因：

1. Timer 依赖 RunLoop。
2. RunLoop 当前正在处理其他耗时任务时，Timer 会延后。
3. RunLoop 不在 Timer 所在 Mode 下时，Timer 不会触发。
4. 主线程繁忙时，Timer 回调会被推迟。

例如：

```objc
[NSThread sleepForTimeInterval:5];
```

如果主线程阻塞 5 秒，即使 Timer 每 1 秒触发一次，也无法准时执行。

### 总结

```text
NSTimer 不是实时定时器
它的触发时机依赖 RunLoop 的状态
```

如果需要更高精度，可以考虑：

```objc
dispatch_source_t
CADisplayLink
```

---

## 19. NSTimer 和 CADisplayLink 的区别？

**答案：**

| 对比项 | NSTimer | CADisplayLink |
|---|---|---|
| 触发依据 | 时间间隔 | 屏幕刷新 |
| 常用场景 | 普通定时任务 | 动画、FPS 监控 |
| 精度 | 受 RunLoop 影响较大 | 和屏幕刷新同步 |
| 默认 Mode | Default | 需要加入 RunLoop |
| 是否适合动画 | 一般 | 适合 |

`CADisplayLink` 每次屏幕刷新时触发，通常一秒 60 次或 120 次，适合动画和 FPS 统计。

---

# 八、Observer 和 RunLoop 状态

## 20. CFRunLoopObserverRef 是什么？

**答案：**

`CFRunLoopObserverRef` 是 RunLoop 的观察者，用来监听 RunLoop 状态变化。

可以监听的状态包括：

```objc
kCFRunLoopEntry
kCFRunLoopBeforeTimers
kCFRunLoopBeforeSources
kCFRunLoopBeforeWaiting
kCFRunLoopAfterWaiting
kCFRunLoopExit
kCFRunLoopAllActivities
```

常见用途：

1. 监听 RunLoop 状态。
2. 实现 AutoreleasePool 释放。
3. 监听主线程卡顿。
4. 观察 UI 刷新时机。

---

## 21. RunLoop 有哪些状态？

**答案：**

常见状态如下：

### 1. kCFRunLoopEntry

即将进入 RunLoop。

### 2. kCFRunLoopBeforeTimers

即将处理 Timer。

### 3. kCFRunLoopBeforeSources

即将处理 Source。

### 4. kCFRunLoopBeforeWaiting

即将进入休眠。

很多 UI 刷新、AutoreleasePool 处理和这个阶段有关。

### 5. kCFRunLoopAfterWaiting

刚从休眠中唤醒。

### 6. kCFRunLoopExit

即将退出 RunLoop。

### 7. kCFRunLoopAllActivities

监听所有状态。

---

## 22. 如何添加 Observer 监听 RunLoop 状态？

**答案：**

Objective-C 示例：

```objc
- (void)addRunLoopObserver {
    CFRunLoopObserverContext context = {
        0,
        (__bridge void *)self,
        NULL,
        NULL,
        NULL
    };

    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(
        kCFAllocatorDefault,
        kCFRunLoopAllActivities,
        YES,
        0,
        runLoopObserverCallBack,
        &context
    );

    CFRunLoopAddObserver(CFRunLoopGetMain(),
                         observer,
                         kCFRunLoopCommonModes);

    CFRelease(observer);
}

static void runLoopObserverCallBack(CFRunLoopObserverRef observer,
                                    CFRunLoopActivity activity,
                                    void *info) {
    switch (activity) {
        case kCFRunLoopEntry:
            NSLog(@"进入 RunLoop");
            break;

        case kCFRunLoopBeforeTimers:
            NSLog(@"即将处理 Timer");
            break;

        case kCFRunLoopBeforeSources:
            NSLog(@"即将处理 Source");
            break;

        case kCFRunLoopBeforeWaiting:
            NSLog(@"即将休眠");
            break;

        case kCFRunLoopAfterWaiting:
            NSLog(@"刚被唤醒");
            break;

        case kCFRunLoopExit:
            NSLog(@"退出 RunLoop");
            break;

        default:
            break;
    }
}
```

---

# 九、RunLoop 内部运行逻辑

## 23. RunLoop 内部大致运行流程是什么？

**答案：**

RunLoop 的内部运行流程可以概括为：

```text
1. 通知 Observer：进入 RunLoop
2. 通知 Observer：即将处理 Timer
3. 通知 Observer：即将处理 Source
4. 处理 Blocks
5. 处理 Source0
6. 如果存在 Source1，跳转到唤醒处理
7. 通知 Observer：即将休眠
8. 线程进入休眠，等待消息唤醒
9. 通知 Observer：已经被唤醒
10. 处理 Timer / GCD 主队列 / Source1
11. 处理 Blocks
12. 根据结果决定继续循环还是退出
13. 通知 Observer：退出 RunLoop
```

可以简化为：

```text
进入循环
处理 Timer
处理 Source
处理 Block
没有任务就休眠
有任务就唤醒
处理唤醒事件
继续循环或退出
```

---

## 24. RunLoop 是怎么响应用户操作的？

**答案：**

以用户点击屏幕为例，流程大致如下：

```text
1. 用户触摸屏幕
2. 系统硬件产生触摸事件
3. IOKit.framework 捕获事件
4. SpringBoard 接收事件
5. 通过 mach port 转发给当前 App
6. App 主线程 RunLoop 被 Source1 唤醒
7. RunLoop 处理系统事件
8. 事件被包装成 UIEvent
9. 通过 Source0 分发给 UIApplication
10. UIApplication 分发给 UIWindow
11. UIWindow 进行 hit-test 找到目标 View
12. 调用 touchesBegan / touchesMoved / touchesEnded
13. 手势识别器参与处理
14. UI 根据事件进行响应
```

### 简化回答

```text
用户操作先由系统捕获，通过 Source1 唤醒主线程 RunLoop，
然后事件被包装并分发到应用层，
最终通过 Source0 触发 UIControl、手势或 touches 方法。
```

---

## 25. GCD 主队列和 RunLoop 有关系吗？

**答案：**

有关系。

当执行：

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"main queue task");
});
```

这个任务会被提交到主队列。  
主线程 RunLoop 在运行过程中会处理主队列中的任务。

在 RunLoop 内部流程中，有一个阶段会处理：

```text
GCD Async Main Queue
```

底层函数中也可以看到类似：

```text
__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__
```

所以主队列任务的执行依赖主线程 RunLoop 的调度。

---

# 十、RunLoop 休眠原理

## 26. RunLoop 是如何休眠和唤醒的？

**答案：**

RunLoop 的休眠和唤醒底层依赖 `mach_msg()`。

大致流程：

```text
用户态调用 mach_msg()
进入内核态等待消息
没有消息时线程休眠
有消息时内核唤醒线程
线程回到用户态处理消息
```

当 RunLoop 没有事件需要处理时，不会一直占用 CPU，而是进入内核态等待。

当以下事件发生时，RunLoop 会被唤醒：

```text
Timer 到时间
Source1 收到消息
GCD 主队列有任务
手势 / 触摸事件
Port 消息
手动唤醒 RunLoop
```

---

## 27. 为什么 RunLoop 能节省 CPU？

**答案：**

因为它不是一直执行空循环。

普通 while 循环：

```objc
while (1) {
    NSLog(@"一直执行");
}
```

这种会持续占用 CPU。

RunLoop：

```text
没有任务时调用 mach_msg 进入休眠
有任务时才被唤醒处理
```

所以 RunLoop 做到了：

```text
该工作时工作
该休息时休息
```

---

# 十一、AutoreleasePool 和 RunLoop

## 28. AutoreleasePool 和 RunLoop 有什么关系？

**答案：**

主线程的 AutoreleasePool 和 RunLoop 有密切关系。

系统会在 RunLoop 中注册 Observer，用来管理 AutoreleasePool。

大致时机：

```text
进入 RunLoop 时创建 AutoreleasePool
即将休眠时释放旧 Pool，并创建新 Pool
退出 RunLoop 时释放 Pool
```

常见状态：

```objc
kCFRunLoopEntry
kCFRunLoopBeforeWaiting
kCFRunLoopExit
```

所以很多 autorelease 对象并不是立刻释放，而是在 RunLoop 某些阶段统一释放。

---

## 29. autorelease 对象什么时候释放？

**答案：**

在主线程中，autorelease 对象通常会在 RunLoop 即将休眠或者退出时释放。

也就是：

```objc
kCFRunLoopBeforeWaiting
kCFRunLoopExit
```

如果在子线程中大量创建 autorelease 对象，最好手动加：

```objc
@autoreleasepool {
    // 大量临时对象
}
```

否则可能导致内存峰值过高。

---

# 十二、UI 刷新和 RunLoop

## 30. UI 刷新和 RunLoop 有什么关系？

**答案：**

iOS 并不是每次调用 `setNeedsLayout` 或 `setNeedsDisplay` 都立即刷新 UI。

例如：

```objc
[self.view setNeedsLayout];
[self.view setNeedsDisplay];
```

这些方法只是做标记。

真正的刷新通常发生在 RunLoop 即将休眠之前：

```objc
kCFRunLoopBeforeWaiting
```

系统会在这个阶段统一处理：

```text
布局更新
约束更新
绘制提交
Core Animation 提交事务
```

这样可以把一次 RunLoop 中多次 UI 修改合并，减少重复刷新，提高性能。

---

## 31. 为什么 setNeedsLayout 不会立即布局？

**答案：**

因为 `setNeedsLayout` 只是标记 View 需要重新布局。

真正调用：

```objc
layoutSubviews
```

通常会等到当前 RunLoop 快结束时统一处理。

这样做的好处是：

```text
合并多次布局请求
减少重复计算
提升性能
```

如果想立即布局，可以调用：

```objc
[self.view layoutIfNeeded];
```

---

# 十三、RunLoop 实际应用

## 32. RunLoop 在实际开发中有哪些应用？

**答案：**

RunLoop 常见应用有：

```text
1. 控制线程生命周期
2. 子线程保活
3. 解决 Timer 滑动时不触发问题
4. 监听主线程卡顿
5. 性能优化
6. UI 刷新时机控制
7. AutoreleasePool 管理
8. 常驻线程处理异步任务
9. 线程间通信
10. FPS 监控
```

---

## 33. 如何用 RunLoop 监控卡顿？

**答案：**

主线程卡顿可以通过监听 RunLoop 状态来判断。

核心思路：

1. 给主线程 RunLoop 添加 Observer。
2. 监听状态变化。
3. 如果 RunLoop 长时间停留在某些状态，说明主线程可能卡住。
4. 使用子线程定时检测主线程状态是否超时。
5. 超过阈值后记录堆栈。

重点关注状态：

```objc
kCFRunLoopBeforeSources
kCFRunLoopAfterWaiting
```

如果主线程长时间停留在这些状态，可能说明正在执行耗时任务。

---

## 34. 卡顿监控的简单实现思路是什么？

**答案：**

示例思路：

```objc
static CFRunLoopActivity runLoopActivity;

static void runLoopObserverCallBack(CFRunLoopObserverRef observer,
                                    CFRunLoopActivity activity,
                                    void *info) {
    runLoopActivity = activity;
}

- (void)startMonitor {
    CFRunLoopObserverContext context = {0, NULL, NULL, NULL, NULL};

    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(
        kCFAllocatorDefault,
        kCFRunLoopAllActivities,
        YES,
        0,
        runLoopObserverCallBack,
        &context
    );

    CFRunLoopAddObserver(CFRunLoopGetMain(),
                         observer,
                         kCFRunLoopCommonModes);

    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES) {
            if (runLoopActivity == kCFRunLoopBeforeSources ||
                runLoopActivity == kCFRunLoopAfterWaiting) {
                // 如果持续超过阈值，例如 80ms、100ms、250ms
                // 可以认为可能发生卡顿
            }

            [NSThread sleepForTimeInterval:0.05];
        }
    });

    CFRelease(observer);
}
```

实际项目中还需要：

```text
信号量判断
超时时间判断
主线程堆栈采集
日志上报
误判过滤
```

---

## 35. RunLoop 如何用于线程保活？

**答案：**

线程默认执行完任务就会销毁。

如果希望一个子线程长期存在，用来不断接收任务，可以启动它的 RunLoop。

关键点：

```text
1. 创建子线程
2. 在线程入口方法中获取 RunLoop
3. 给 RunLoop 添加 Source / Port
4. 启动 RunLoop
5. 后续通过 performSelector:onThread: 投递任务
```

核心代码：

```objc
- (void)run {
    @autoreleasepool {
        [[NSRunLoop currentRunLoop] addPort:[NSPort port]
                                    forMode:NSDefaultRunLoopMode];

        [[NSRunLoop currentRunLoop] run];
    }
}
```

但是更严谨的写法应该支持停止线程：

```objc
while (!self.stopped) {
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                             beforeDate:[NSDate distantFuture]];
}
```

---

# 十四、容易混淆的问题

## 36. RunLoop 一定会一直运行吗？

**答案：**

不一定。

RunLoop 要运行，需要满足条件：

```text
当前 Mode 中至少有 Source、Timer、Observer 中的一种
```

如果当前 Mode 中什么都没有，RunLoop 会直接退出。

例如：

```objc
[[NSRunLoop currentRunLoop] run];
```

如果没有添加任何事件源，它不会一直运行。

---

## 37. RunLoop 可以切换 Mode 吗？

**答案：**

可以，但不是在当前循环中直接切换。

规则是：

```text
RunLoop 启动时只能选择一个 Mode
如果要切换 Mode，需要退出当前 Loop
然后重新指定另一个 Mode 进入
```

例如从默认模式切换到滑动模式：

```text
NSDefaultRunLoopMode
退出
UITrackingRunLoopMode
进入
```

---

## 38. NSRunLoopCommonModes 是 Mode 吗？

**答案：**

不是。

`NSRunLoopCommonModes` 不是一个真正的运行模式，而是一个集合标记。

它表示：

```text
把某个 Timer / Source 添加到 Common Modes 包含的所有 Mode 中
```

通常包括：

```objc
NSDefaultRunLoopMode
UITrackingRunLoopMode
```

所以：

```objc
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

意思不是 Timer 运行在一个叫 Common 的 Mode 里，而是 Timer 会被添加到 Common Modes 标记下的真实 Mode 中。

---

## 39. performSelector:afterDelay: 和 RunLoop 有关系吗？

**答案：**

有关系。

例如：

```objc
[self performSelector:@selector(test)
           withObject:nil
           afterDelay:2.0];
```

它底层依赖 Timer，也就是依赖 RunLoop。

如果当前线程没有启动 RunLoop，这个方法可能不会执行。

例如在子线程中：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [self performSelector:@selector(test)
               withObject:nil
               afterDelay:2.0];
});
```

这段代码可能不会执行 `test`，因为子线程默认没有运行 RunLoop。

需要这样：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [self performSelector:@selector(test)
               withObject:nil
               afterDelay:2.0];

    [[NSRunLoop currentRunLoop] run];
});
```

---

## 40. performSelector:onThread: 和 RunLoop 有关系吗？

**答案：**

有关系。

例如：

```objc
[self performSelector:@selector(task)
             onThread:self.thread
           withObject:nil
        waitUntilDone:NO];
```

这个方法需要目标线程的 RunLoop 正在运行，否则任务无法被处理。

所以常驻线程通常会配合：

```objc
performSelector:onThread:
```

使用。

---

# 十五、综合题

## 41. 讲讲 RunLoop，项目中一般怎么用？

**答案参考：**

RunLoop 是 iOS 中线程的事件循环机制。每个线程都有唯一对应的 RunLoop，主线程的 RunLoop 默认开启，子线程默认不开启。

RunLoop 的作用主要是让线程在有任务时执行任务，没有任务时进入休眠，从而避免线程退出并节省 CPU。

它内部主要由 Mode、Source、Timer、Observer 组成。一个 RunLoop 可以有多个 Mode，但每次只能运行在一个 Mode 下。每个 Mode 中可以包含 Source0、Source1、Timer 和 Observer。

实际项目中常见应用有：

```text
1. 使用 NSTimer 时处理滑动列表导致 Timer 暂停的问题
2. 子线程保活
3. 监控主线程卡顿
4. 观察 RunLoop 状态
5. 理解 UI 刷新时机
6. 理解 AutoreleasePool 释放时机
```

例如 Timer 默认加入 `NSDefaultRunLoopMode`，当用户滑动 `UITableView` 时，RunLoop 会切换到 `UITrackingRunLoopMode`，导致 Timer 暂停。解决方式是把 Timer 添加到 `NSRunLoopCommonModes`。

---

## 42. 详细说说 RunLoop 的内部实现逻辑。

**答案参考：**

RunLoop 内部是一个事件循环。

大致流程如下：

```text
1. 通知 Observer：进入 RunLoop
2. 通知 Observer：即将处理 Timer
3. 通知 Observer：即将处理 Source
4. 处理 RunLoop Blocks
5. 处理 Source0
6. 如果有 Source1，处理 Source1
7. 没有任务时，通知 Observer 即将休眠
8. 调用 mach_msg 进入内核态休眠
9. 有消息时被唤醒
10. 通知 Observer 已经唤醒
11. 根据唤醒原因处理 Timer、GCD 主队列任务、Source1
12. 再次处理 Blocks
13. 判断继续循环还是退出
14. 通知 Observer 退出 RunLoop
```

它的关键点是：

```text
有事做事
没事休眠
有消息再唤醒
```

底层通过 `mach_msg()` 实现线程休眠和唤醒。

---

## 43. RunLoop 和线程的关系是什么？

**答案参考：**

每个线程都有唯一对应的 RunLoop。

RunLoop 存在于一个全局字典中：

```text
key: pthread_t
value: CFRunLoopRef
```

线程刚创建时，并不会自动创建 RunLoop。  
当第一次调用：

```objc
[NSRunLoop currentRunLoop]
```

或者：

```objc
CFRunLoopGetCurrent()
```

时，系统才会为当前线程创建 RunLoop。

主线程的 RunLoop 默认已经创建并启动。  
子线程的 RunLoop 默认不会自动运行，需要手动启动。

线程销毁时，对应的 RunLoop 也会销毁。

---

## 44. Timer 和 RunLoop 的关系是什么？

**答案参考：**

Timer 依赖 RunLoop 工作。

`NSTimer` 底层对应的是：

```objc
CFRunLoopTimerRef
```

Timer 必须添加到 RunLoop 的某个 Mode 中，只有 RunLoop 当前运行的 Mode 和 Timer 所在 Mode 一致时，Timer 才会触发。

例如：

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:1
                                         target:self
                                       selector:@selector(test)
                                       userInfo:nil
                                        repeats:YES];

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
```

如果主线程正在滑动列表，RunLoop 处于 `UITrackingRunLoopMode`，这个 Timer 就不会触发。

解决方式：

```objc
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

---

## 45. 程序中添加每 3 秒响应一次的 NSTimer，当拖动 TableView 时 Timer 无法响应，怎么解决？

**答案参考：**

原因是 Timer 默认运行在：

```objc
NSDefaultRunLoopMode
```

当拖动 TableView 时，主线程 RunLoop 会切换到：

```objc
UITrackingRunLoopMode
```

这时默认模式下的 Timer 不会被处理。

解决方式是把 Timer 添加到：

```objc
NSRunLoopCommonModes
```

示例：

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:3.0
                                         target:self
                                       selector:@selector(timerAction)
                                       userInfo:nil
                                        repeats:YES];

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

Swift：

```swift
let timer = Timer(timeInterval: 3.0, repeats: true) { _ in
    print("timer fired")
}

RunLoop.main.add(timer, forMode: .common)
```

这样 Timer 在默认状态和滑动状态下都可以被触发。

---

## 46. RunLoop 是怎么响应用户操作的？

**答案参考：**

用户触摸屏幕后，系统会捕获硬件事件，然后通过 mach port 把事件发送到当前 App。

主线程 RunLoop 被 Source1 唤醒，然后事件被包装成 `UIEvent`，再通过 Source0 分发给应用层。

大致流程：

```text
用户触摸屏幕
系统捕获事件
mach port 发送给 App
Source1 唤醒 RunLoop
RunLoop 处理系统事件
包装成 UIEvent
UIApplication 分发事件
UIWindow 进行 hit-test
找到目标 View
调用 touches 或手势回调
```

所以可以总结为：

```text
Source1 负责接收系统事件并唤醒 RunLoop
Source0 负责在 App 内部分发和处理事件
```

---

## 47. 说说 RunLoop 的几种状态。

**答案参考：**

RunLoop 常见状态有：

```objc
kCFRunLoopEntry
```

即将进入 RunLoop。

```objc
kCFRunLoopBeforeTimers
```

即将处理 Timer。

```objc
kCFRunLoopBeforeSources
```

即将处理 Source。

```objc
kCFRunLoopBeforeWaiting
```

即将进入休眠。

```objc
kCFRunLoopAfterWaiting
```

刚从休眠中被唤醒。

```objc
kCFRunLoopExit
```

即将退出 RunLoop。

```objc
kCFRunLoopAllActivities
```

监听所有状态。

这些状态可以通过 `CFRunLoopObserverRef` 监听。

---

## 48. RunLoop 的 Mode 作用是什么？

**答案参考：**

Mode 用来隔离不同类型的事件源。

一个 RunLoop 可以有多个 Mode，每个 Mode 中有自己的 Source、Timer、Observer。

RunLoop 每次只能运行在一个 Mode 下，因此不同 Mode 中的事件互不影响。

例如：

```text
NSDefaultRunLoopMode：默认模式
UITrackingRunLoopMode：滑动追踪模式
```

当用户滑动 ScrollView 时，RunLoop 切换到 `UITrackingRunLoopMode`，这样可以保证滑动事件优先处理，避免被默认模式中的 Timer 等事件干扰。

---

# 十六、加分回答模板

## 49. 用一句话总结 RunLoop

**答案：**

RunLoop 是线程的事件循环机制，它让线程在有任务时处理任务，没有任务时休眠，从而保持线程存活并节省 CPU。

---

## 50. 用一段完整话术回答 RunLoop

**答案：**

RunLoop 是 iOS 中非常核心的事件循环机制。每个线程都有唯一对应的 RunLoop，主线程的 RunLoop 默认已经创建并运行，子线程默认不会自动开启。RunLoop 内部包含多个 Mode，每个 Mode 中又包含 Source、Timer 和 Observer。RunLoop 每次只能运行在一个 Mode 下，通过 Mode 可以隔离不同类型的事件。比如默认情况下主线程运行在 `NSDefaultRunLoopMode`，当用户滑动 ScrollView 时会切换到 `UITrackingRunLoopMode`，所以默认 Mode 下的 Timer 在滑动时不会触发。RunLoop 的底层通过 `mach_msg()` 实现休眠和唤醒，有任务时处理任务，没有任务时进入休眠。实际项目中，RunLoop 常用于 Timer 问题处理、线程保活、卡顿监控、UI 刷新时机分析和 AutoreleasePool 释放时机理解。

---

# 十七、速记版

```text
RunLoop 是线程的事件循环
每条线程都有唯一 RunLoop
主线程 RunLoop 默认开启
子线程 RunLoop 默认不开启
RunLoop 保存在全局字典中
RunLoop 包含多个 Mode
每次只能运行一个 Mode
Mode 包含 Source、Timer、Observer
Source0 处理触摸、performSelector
Source1 处理 Port、系统事件、线程通信
Timer 依赖 RunLoop 和 Mode
Observer 监听 RunLoop 状态
DefaultMode 是默认模式
UITrackingMode 是滑动模式
CommonModes 不是模式，是集合标记
Timer 滑动不走是因为 Mode 切换
解决方案是添加到 CommonModes
RunLoop 底层通过 mach_msg 休眠和唤醒
UI 刷新通常在 BeforeWaiting
AutoreleasePool 也和 RunLoop Observer 有关
卡顿监控可以监听 RunLoop 状态
线程保活需要启动子线程 RunLoop 并添加 Source
```

---

# 十八、最容易答错的点

## 1. CommonModes 不是一个真正的 Mode

错误说法：

```text
Timer 加到了 CommonModes 模式
```

正确说法：

```text
CommonModes 是一个 Mode 集合标记，Timer 会被添加到 CommonModes 包含的真实 Mode 中
```

---

## 2. 子线程不是没有 RunLoop

错误说法：

```text
子线程没有 RunLoop
```

正确说法：

```text
子线程有对应 RunLoop 的能力，但默认不会创建和运行。第一次获取时创建，需要手动启动。
```

---

## 3. Timer 不是绝对精准

错误说法：

```text
NSTimer 每秒一定准时触发
```

正确说法：

```text
NSTimer 依赖 RunLoop，如果线程忙、Mode 不匹配或 RunLoop 阻塞，Timer 会延迟。
```

---

## 4. UI 不是调用 setNeedsDisplay 后立即刷新

错误说法：

```text
setNeedsDisplay 会立即绘制
```

正确说法：

```text
setNeedsDisplay 只是标记需要绘制，真正绘制通常在 RunLoop 即将休眠前统一提交。
```

---

## 5. RunLoop 不是一直占用 CPU

错误说法：

```text
RunLoop 是死循环，所以一直占 CPU
```

正确说法：

```text
RunLoop 没任务时会通过 mach_msg 进入休眠，不会一直占用 CPU。
```

---

# 十九、建议背诵的核心答案

你可以重点背下面这一段：

> RunLoop 是线程的事件循环机制，每条线程都有唯一对应的 RunLoop。主线程的 RunLoop 默认开启，子线程默认不开启。RunLoop 内部包含多个 Mode，每个 Mode 中有 Source、Timer、Observer。RunLoop 每次只能运行在一个 Mode 下，Mode 用来隔离不同类型的事件。比如 Timer 默认在 DefaultMode 下，当用户滑动 ScrollView 时，RunLoop 会切换到 UITrackingRunLoopMode，所以 Timer 不会触发，解决方式是把 Timer 添加到 CommonModes。RunLoop 底层通过 mach_msg 实现休眠和唤醒，有任务时处理任务，没有任务时休眠。实际项目中常用于 Timer 问题、线程保活、卡顿监控、UI 刷新和 AutoreleasePool 释放时机分析。
