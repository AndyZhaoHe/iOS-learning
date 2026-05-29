# iOS 内存管理高频问答详解

> 来源：`05-内存管理.pptx` 主题内容整理。  
> 适合用于系统复习：定时器循环引用、内存布局、Tagged Pointer、引用计数、ARC、weak、copy、autorelease pool、RunLoop 与自动释放池等。

---

## 目录

1. CADisplayLink、NSTimer 使用注意点
2. iOS 程序内存布局
3. Tagged Pointer 原理
4. OC 对象引用计数机制
5. ARC 做了什么
6. weak 指针实现原理
7. copy 与 mutableCopy
8. 引用计数存储位置
9. dealloc 调用流程
10. 自动释放池原理
11. AutoreleasePoolPage 结构
12. RunLoop 与 AutoreleasePool
13. 局部对象出了方法是否立即释放
14. 高频追问总结
15. 综合代码题

---

# 一、CADisplayLink、NSTimer 使用注意点

## 1. 使用 CADisplayLink、NSTimer 有什么注意点？

### 答案

`CADisplayLink` 和 `NSTimer` 最大的问题是：**它们会对 target 产生强引用**。

如果控制器强引用了定时器，而定时器又强引用了控制器，就会形成循环引用：

```objc
@interface ViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                  target:self
                                                selector:@selector(timerAction)
                                                userInfo:nil
                                                 repeats:YES];
}

- (void)timerAction {
    NSLog(@"timer running");
}

- (void)dealloc {
    NSLog(@"ViewController dealloc");
}

@end
```

上面代码中：

```text
ViewController -> strong -> NSTimer
NSTimer -> strong -> target(ViewController)
```

结果就是 `ViewController` 无法释放，`dealloc` 不会调用。

---

## 2. NSTimer 为什么容易造成循环引用？

### 答案

因为 `NSTimer` 被加入 RunLoop 后，RunLoop 会持有 Timer；Timer 又会持有它的 target。

引用关系大致如下：

```text
RunLoop -> Timer -> Target(ViewController)
ViewController -> Timer
```

如果 Timer 是重复执行的，即 `repeats:YES`，它不会自动失效，所以这个引用链会一直存在。

---

## 3. 如何解决 NSTimer 循环引用？

### 方案一：使用 block，并弱引用 self

```objc
__weak typeof(self) weakSelf = self;
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) {
        [timer invalidate];
        return;
    }
    [strongSelf timerAction];
}];
```

注意：

- block 会持有 block 内部捕获的对象。
- 所以 block 内不能直接使用 `self`。
- 应该使用 `weakSelf` 避免循环引用。

---

### 方案二：使用代理对象

可以创建一个中间对象，让 Timer 强引用中间对象，而不是直接强引用控制器。

```objc
@interface WeakProxy : NSProxy
@property (nonatomic, weak) id target;
+ (instancetype)proxyWithTarget:(id)target;
@end

@implementation WeakProxy

+ (instancetype)proxyWithTarget:(id)target {
    WeakProxy *proxy = [WeakProxy alloc];
    proxy.target = target;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [self.target methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    if ([self.target respondsToSelector:invocation.selector]) {
        [invocation invokeWithTarget:self.target];
    }
}

@end
```

使用：

```objc
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                              target:[WeakProxy proxyWithTarget:self]
                                            selector:@selector(timerAction)
                                            userInfo:nil
                                             repeats:YES];
```

引用关系变成：

```text
ViewController -> Timer -> WeakProxy --weak--> ViewController
```

这样就不会形成循环引用。

---

### 方案三：手动 invalidate

```objc
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    [self.timer invalidate];
    self.timer = nil;
}
```

但是这不是最稳妥的方案，因为如果某些情况下没有走到 `viewWillDisappear`，仍然可能泄漏。

---

## 4. CADisplayLink 和 NSTimer 有什么区别？

### 答案

| 对比项 | NSTimer | CADisplayLink |
|---|---|---|
| 触发依据 | 时间间隔 | 屏幕刷新频率 |
| 常见用途 | 普通定时任务 | 动画、UI 刷新 |
| 是否依赖 RunLoop | 是 | 是 |
| 是否强引用 target | 是 | 是 |
| 准确性 | 受 RunLoop 影响 | 与屏幕刷新同步 |

`CADisplayLink` 通常每次屏幕刷新时触发一次，比如 60Hz 屏幕大约每秒触发 60 次。

---

## 5. NSTimer 为什么可能不准？

### 答案

`NSTimer` 依赖 RunLoop。

如果主线程 RunLoop 正在处理大量任务，例如：

- UI 渲染卡顿
- 大量计算
- 滚动时 RunLoop Mode 切换
- 主线程阻塞

那么 Timer 的触发时间就会被延迟。

例如：

```objc
[NSTimer scheduledTimerWithTimeInterval:1.0
                                 target:self
                               selector:@selector(action)
                               userInfo:nil
                                repeats:YES];
```

这个 Timer 并不保证每 1 秒绝对准时触发，只能保证 RunLoop 有机会处理 Timer 时触发。

---

## 6. GCD 定时器为什么更准？

### 答案

GCD 定时器基于系统内核调度，不依赖 RunLoop，所以在很多情况下比 `NSTimer` 更准。

示例：

```objc
@property (nonatomic, strong) dispatch_source_t gcdTimer;

- (void)startGCDTimer {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    self.gcdTimer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    
    dispatch_source_set_timer(self.gcdTimer,
                              dispatch_time(DISPATCH_TIME_NOW, 0),
                              1.0 * NSEC_PER_SEC,
                              0.1 * NSEC_PER_SEC);
    
    dispatch_source_set_event_handler(self.gcdTimer, ^{
        NSLog(@"GCD timer fire");
    });
    
    dispatch_resume(self.gcdTimer);
}
```

注意：

- GCD Timer 需要强引用保存，否则会被释放。
- 不需要时要取消：

```objc
if (self.gcdTimer) {
    dispatch_source_cancel(self.gcdTimer);
    self.gcdTimer = nil;
}
```

---

# 二、iOS 程序内存布局

## 7. iOS 程序的内存布局是怎样的？

### 答案

iOS 程序的内存大致可以分为以下几个区域：

```text
低地址

代码段（__TEXT）
数据段（__DATA）
  - 字符串常量
  - 已初始化数据
  - 未初始化数据
堆（heap） ↓
栈（stack） ↑
内核区

高地址
```

---

## 8. 代码段存放什么？

### 答案

代码段也叫 `__TEXT` 段，主要存放编译之后的机器指令。

例如：

```objc
- (void)test {
    NSLog(@"hello");
}
```

这个方法编译后的机器代码会放在代码段中。

特点：

- 通常只读
- 存放程序执行指令
- 多个进程可以共享同一份代码段

---

## 9. 数据段存放什么？

### 答案

数据段通常包括：

1. 字符串常量
2. 已初始化的全局变量
3. 已初始化的静态变量
4. 未初始化的全局变量
5. 未初始化的静态变量

示例：

```objc
NSString *globalName = @"PitPat";       // 全局变量
static int age = 10;                    // 已初始化静态变量
static int count;                       // 未初始化静态变量
```

其中：

```objc
NSString *str = @"123";
```

`@"123"` 这种字符串常量一般存放在数据段中的常量区。

---

## 10. 堆和栈有什么区别？

### 答案

| 对比项 | 栈 | 堆 |
|---|---|---|
| 管理方式 | 系统自动管理 | 程序员或 ARC 管理 |
| 存储内容 | 局部变量、函数调用信息 | 动态创建的对象 |
| 分配速度 | 快 | 相对慢 |
| 空间大小 | 较小 | 较大 |
| 地址增长方向 | 从高地址向低地址 | 从低地址向高地址 |
| 生命周期 | 函数结束自动回收 | 引用计数为 0 才释放 |

示例：

```objc
- (void)test {
    int age = 10;                     // age 在栈上
    NSObject *obj = [[NSObject alloc] init]; // obj 指针变量在栈上，对象本体在堆上
}
```

注意：

- `obj` 是局部指针变量，存在栈上。
- `[[NSObject alloc] init]` 创建出来的对象本体存在堆上。

---

# 三、Tagged Pointer 原理

## 11. 什么是 Tagged Pointer？

### 答案

`Tagged Pointer` 是 Apple 在 64 位系统中引入的一种小对象优化技术。

它主要用于优化一些小对象，例如：

- NSNumber
- NSDate
- NSString 的部分小字符串

在没有 Tagged Pointer 之前：

```objc
NSNumber *number = @(10);
```

`number` 指针保存的是堆中 NSNumber 对象的地址。

使用 Tagged Pointer 后：

```text
指针本身 = Tag + Data
```

也就是说，数据直接存储在指针变量里面，不一定需要在堆上创建对象。

---

## 12. Tagged Pointer 有什么好处？

### 答案

好处主要有：

1. 减少堆内存分配
2. 减少对象创建和销毁成本
3. 减少引用计数维护成本
4. 提高访问速度

普通对象：

```text
指针 -> 堆对象 -> 数据
```

Tagged Pointer：

```text
指针本身直接包含数据
```

---

## 13. objc_msgSend 如何处理 Tagged Pointer？

### 答案

`objc_msgSend` 可以识别 Tagged Pointer。

例如：

```objc
NSNumber *num = @(10);
NSLog(@"%d", num.intValue);
```

如果 `num` 是 Tagged Pointer，`objc_msgSend` 不需要真的从堆对象中取值，而是可以根据指针里的 tag 和 data 直接解析出数据。

---

## 14. 如何判断一个指针是不是 Tagged Pointer？

### 答案

根据平台不同判断方式不同：

| 平台 | 判断方式 |
|---|---|
| iOS | 最高有效位是 1 |
| Mac | 最低有效位是 1 |

示例：

```objc
NSNumber *num1 = @(10);
NSNumber *num2 = @(0xFFFFFFFFFFFFFFF);
NSLog(@"%p", num1);
NSLog(@"%p", num2);
```

较小的数字更可能使用 Tagged Pointer；过大的数字无法直接塞进指针，就会退化为普通堆对象。

---

## 15. Tagged Pointer 对引用计数有什么影响？

### 答案

Tagged Pointer 不是真正意义上的堆对象，因此通常不需要维护引用计数。

例如：

```objc
NSNumber *num = @(10);
```

如果它是 Tagged Pointer，那么：

- retain/release 基本不产生普通对象那种引用计数变化
- 不需要 dealloc
- 不需要从堆上释放内存

这也是它性能更好的原因。

---

# 四、OC 对象引用计数机制

## 16. iOS 中 OC 对象是如何管理内存的？

### 答案

OC 对象主要通过引用计数管理内存。

规则：

- 新创建对象引用计数默认为 1
- `retain` 引用计数 +1
- `release` 引用计数 -1
- 引用计数减为 0，对象销毁
- 对象销毁前会调用 `dealloc`

示例：

```objc
Person *p = [[Person alloc] init]; // 引用计数 = 1
[p retain];                       // 引用计数 = 2
[p release];                      // 引用计数 = 1
[p release];                      // 引用计数 = 0，调用 dealloc
```

---

## 17. MRC 下内存管理原则是什么？

### 答案

MRC 下有一条非常重要的规则：

只要你通过以下方法获得对象所有权，就需要在不使用时释放：

- alloc
- new
- copy
- mutableCopy
- retain

口诀：

```text
谁创建，谁释放。
谁持有，谁释放。
```

示例：

```objc
Person *p = [[Person alloc] init];
// 使用 p
[p release];
```

如果使用了 `autorelease`：

```objc
Person *p = [[[Person alloc] init] autorelease];
```

那么对象会被加入自动释放池，稍后由自动释放池统一发送 `release`。

---

## 18. retain、release、autorelease 分别做什么？

### 答案

### retain

```objc
[p retain];
```

表示当前对象想继续持有 `p`，引用计数 +1。

### release

```objc
[p release];
```

表示当前对象不再持有 `p`，引用计数 -1。

### autorelease

```objc
[p autorelease];
```

表示当前不立即释放，而是把对象加入自动释放池，等池子销毁时再发送 `release`。

---

## 19. 什么情况下会调用 dealloc？

### 答案

当对象引用计数变为 0 时，会自动调用 `dealloc`。

示例：

```objc
- (void)dealloc {
    NSLog(@"Person dealloc");
    [super dealloc]; // MRC 下需要调用，ARC 下不允许调用
}
```

注意：

- ARC 下可以重写 `dealloc` 做资源清理。
- ARC 下不能手动调用 `[super dealloc]`。
- MRC 下必须调用 `[super dealloc]`。

---

# 五、ARC 做了什么

## 20. ARC 是什么？

### 答案

ARC 是 Automatic Reference Counting，自动引用计数。

它并不是 GC，也不是运行时自动扫描内存。

ARC 的核心是：

```text
编译器在合适的位置自动插入 retain、release、autorelease 等内存管理代码。
```

也就是说，ARC 主要是编译期技术，由 LLVM 完成大部分工作，同时配合 Runtime。

---

## 21. ARC 都帮我们做了什么？

### 答案

ARC 会帮我们：

1. 在对象需要被持有时插入 retain
2. 在对象不再使用时插入 release
3. 根据返回值约定插入 autorelease 或优化掉 autorelease
4. 处理 strong、weak、copy 等属性语义
5. 在对象销毁时自动清理强引用成员变量
6. 在对象销毁时配合 Runtime 清理 weak 引用

例如：

```objc
- (void)test {
    Person *p = [[Person alloc] init];
    NSLog(@"%@", p);
}
```

在 ARC 下，编译器会在适当位置自动插入释放逻辑。开发者不需要手写：

```objc
[p release];
```

---

## 22. ARC 和 GC 有什么区别？

### 答案

| 对比项 | ARC | GC |
|---|---|---|
| 全称 | 自动引用计数 | 垃圾回收 |
| 工作时机 | 编译期为主 | 运行期为主 |
| 是否基于引用计数 | 是 | 不一定 |
| 是否需要扫描对象图 | 不需要全局扫描 | 通常需要 |
| 是否有循环引用问题 | 有 | 多数 GC 可处理循环引用 |
| iOS 是否使用 | 是 | 否 |

ARC 仍然是引用计数，所以无法自动解决强引用循环。

---

## 23. ARC 能解决循环引用吗？

### 答案

不能。

例如：

```objc
@interface Person : NSObject
@property (nonatomic, strong) Dog *dog;
@end

@interface Dog : NSObject
@property (nonatomic, strong) Person *owner;
@end
```

如果：

```objc
person.dog = dog;
dog.owner = person;
```

引用关系：

```text
Person -> Dog
Dog -> Person
```

两个对象互相强引用，引用计数都无法变为 0，因此都不会释放。

解决方式：

```objc
@property (nonatomic, weak) Person *owner;
```

---

# 六、weak 指针实现原理

## 24. weak 的作用是什么？

### 答案

`weak` 用来修饰弱引用。

特点：

1. 不增加对象引用计数
2. 对象释放后，weak 指针自动置为 nil
3. 常用于解决循环引用

示例：

```objc
@property (nonatomic, weak) id delegate;
```

delegate 通常用 weak，因为代理对象一般不应该被被代理对象强持有。

---

## 25. weak 指针的实现原理是什么？

### 答案

weak 的底层依赖 Runtime 的弱引用表。

大致流程：

1. 当一个对象被 weak 指针引用时，Runtime 会把这个 weak 指针地址登记到弱引用表中。
2. 弱引用表通常以对象地址为 key。
3. value 是所有指向该对象的 weak 指针地址集合。
4. 当对象释放时，Runtime 会根据对象地址找到所有 weak 指针。
5. 然后把这些 weak 指针全部置为 nil。

可以理解为：

```text
SideTable
  weak_table
    objectAddress -> [weakPtr1, weakPtr2, weakPtr3]
```

对象释放时：

```text
遍历 weakPtr 数组，把每个 weakPtr 指向的内容置 nil
```

---

## 26. weak 为什么能自动置 nil？

### 答案

因为 weak 指针并不是单纯地保存对象地址。

它在赋值时会经过 Runtime 函数处理，例如类似：

```objc
objc_storeWeak(&weakPtr, obj);
```

Runtime 会记录：

```text
哪个 weak 变量指向了哪个对象
```

当对象销毁时，再通过弱引用表找到这些变量，把它们清空。

---

## 27. weak 和 assign 有什么区别？

### 答案

| 对比项 | weak | assign |
|---|---|---|
| 是否增加引用计数 | 否 | 否 |
| 对象释放后是否自动置 nil | 是 | 否 |
| 是否安全 | 较安全 | 容易野指针 |
| 常见用途 | delegate、避免循环引用 | 基本数据类型 |

错误示例：

```objc
@property (nonatomic, assign) NSObject *obj;
```

如果 `obj` 指向的对象被释放，`obj` 仍然保存旧地址，再访问就可能崩溃。

---

## 28. weak 和 unsafe_unretained 有什么区别？

### 答案

`unsafe_unretained` 和 `assign` 类似，不增加引用计数，也不会自动置 nil。

| 对比项 | weak | unsafe_unretained |
|---|---|---|
| 是否强持有对象 | 否 | 否 |
| 对象释放后 | 自动 nil | 变成野指针 |
| 安全性 | 高 | 低 |
| 性能 | 稍低 | 稍高 |

现代开发中，除非非常特殊的性能场景，一般不建议使用 `unsafe_unretained`。

---

# 七、copy 与 mutableCopy

## 29. copy 和 mutableCopy 有什么区别？

### 答案

`copy` 返回不可变对象，`mutableCopy` 返回可变对象。

| 方法 | 返回对象 |
|---|---|
| copy | 不可变副本 |
| mutableCopy | 可变副本 |

例如：

```objc
NSString *str = @"abc";
NSString *copyStr = [str copy];
NSMutableString *mutableStr = [str mutableCopy];
```

`copyStr` 是不可变字符串，`mutableStr` 是可变字符串。

---

## 30. NSString 调用 copy 是浅拷贝还是深拷贝？

### 答案

不可变对象调用 `copy` 通常是浅拷贝。

```objc
NSString *str1 = @"abc";
NSString *str2 = [str1 copy];

NSLog(@"%p", str1);
NSLog(@"%p", str2);
```

很多情况下，两个地址相同。

原因是：

- 原对象已经不可变
- copy 后也要求不可变
- 没必要重新创建对象

所以直接返回原对象并增加持有关系即可。

---

## 31. NSMutableString 调用 copy 是浅拷贝还是深拷贝？

### 答案

可变对象调用 `copy` 是深拷贝，并且返回不可变对象。

```objc
NSMutableString *str1 = [NSMutableString stringWithString:@"abc"];
NSString *str2 = [str1 copy];

[str1 appendString:@"123"];

NSLog(@"%@", str1); // abc123
NSLog(@"%@", str2); // abc
```

因为如果直接返回同一个对象，那么原可变字符串变化后，会影响 copy 出来的对象，这不符合 copy 语义。

---

## 32. copy 和 mutableCopy 总结表

### 答案

| 原对象 | copy 结果 | copy 类型 | mutableCopy 结果 | mutableCopy 类型 |
|---|---|---|---|---|
| NSString | NSString | 浅拷贝 | NSMutableString | 深拷贝 |
| NSMutableString | NSString | 深拷贝 | NSMutableString | 深拷贝 |
| NSArray | NSArray | 浅拷贝 | NSMutableArray | 深拷贝 |
| NSMutableArray | NSArray | 深拷贝 | NSMutableArray | 深拷贝 |
| NSDictionary | NSDictionary | 浅拷贝 | NSMutableDictionary | 深拷贝 |
| NSMutableDictionary | NSDictionary | 深拷贝 | NSMutableDictionary | 深拷贝 |

口诀：

```text
不可变对象 copy 通常浅拷贝。
可变对象 copy 通常深拷贝。
mutableCopy 通常都是深拷贝。
copy 返回不可变对象。
mutableCopy 返回可变对象。
```

---

## 33. 为什么 NSString 属性经常使用 copy？

### 答案

为了防止外部传入 NSMutableString 后被外部修改。

错误写法：

```objc
@property (nonatomic, strong) NSString *name;
```

示例：

```objc
NSMutableString *str = [NSMutableString stringWithString:@"Jack"];
person.name = str;
[str appendString:@" Rose"];
NSLog(@"%@", person.name); // 可能变成 Jack Rose
```

如果使用 copy：

```objc
@property (nonatomic, copy) NSString *name;
```

赋值时会生成不可变副本，外部再修改原可变字符串，不会影响 `person.name`。

---

# 八、引用计数存储位置

## 34. 引用计数存在哪里？

### 答案

在 64 位系统中，引用计数可能存放在两个地方：

1. 优化过的 isa 指针中
2. SideTable 中

也就是说，对象的引用计数不一定单独存在对象内部某个普通成员变量里。

---

## 35. 什么情况下引用计数存储在 isa 中？

### 答案

现代 Objective-C 对象的 isa 是经过优化的，叫做 nonpointer isa。

isa 指针中除了存储类信息，还可以用部分 bit 位存储：

- 引用计数相关信息
- 是否有关联对象
- 是否有 C++ 析构函数
- 是否正在释放
- 是否使用 SideTable

当引用计数较小、能放进 isa 的 bit 位中时，可以直接存在 isa 中。

---

## 36. 什么是 SideTable？

### 答案

SideTable 是 Runtime 用来辅助对象管理的数据结构。

它里面通常包含：

1. 自旋锁或互斥锁
2. 引用计数表
3. weak 弱引用表

可以理解为：

```text
SideTable
  - spinlock_t slock
  - RefcountMap refcnts
  - weak_table_t weak_table
```

其中 `refcnts` 是一个散列表，用于存储对象引用计数。

---

# 九、dealloc 调用流程

## 37. 对象释放时 dealloc 的调用流程是什么？

### 答案

当对象引用计数变为 0 后，会触发释放流程。

大致调用轨迹：

```text
dealloc
_objc_rootDealloc
rootDealloc
object_dispose
objc_destructInstance
free
```

说明：

1. `dealloc`：对象释放入口。
2. `_objc_rootDealloc`：Runtime 根释放逻辑。
3. `rootDealloc`：判断是否可以快速释放。
4. `object_dispose`：销毁对象实例。
5. `objc_destructInstance`：清理成员变量、关联对象、weak 表等。
6. `free`：释放堆内存。

---

## 38. dealloc 中通常做什么？

### 答案

通常做资源清理，例如：

- 移除通知
- 停止定时器
- 关闭文件
- 取消网络请求
- 释放 C/C++ 资源
- 移除 KVO

示例：

```objc
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    [self.timer invalidate];
    NSLog(@"%@ dealloc", self);
}
```

ARC 下不需要释放 strong 属性，因为 ARC 会自动处理。

---

# 十、自动释放池原理

## 39. 什么是自动释放池？

### 答案

自动释放池用于延迟释放对象。

当对象调用 `autorelease` 后，不会马上 release，而是被加入当前自动释放池。

等自动释放池销毁时，会统一向池中的对象发送 `release`。

示例：

```objc
@autoreleasepool {
    Person *p = [[[Person alloc] init] autorelease];
}
// 离开 autoreleasepool 后，p 收到 release
```

---

## 40. @autoreleasepool 的底层是什么？

### 答案

`@autoreleasepool` 编译后会被转换成类似下面的结构：

```objc
void *ctx = objc_autoreleasePoolPush();
// 中间代码
objc_autoreleasePoolPop(ctx);
```

也就是说：

```objc
@autoreleasepool {
    // code
}
```

底层大致等价于：

```objc
void *pool = objc_autoreleasePoolPush();
// code
objc_autoreleasePoolPop(pool);
```

---

## 41. autorelease 对象什么时候会调用 release？

### 答案

autorelease 对象会在自动释放池 pop 时收到 release。

常见时机：

1. 手动 `@autoreleasepool` 作用域结束
2. 主线程 RunLoop 即将休眠时
3. 主线程 RunLoop 即将退出时
4. 子线程自己创建的 autoreleasepool 结束时

---

## 42. 自动释放池的底层数据结构是什么？

### 答案

主要是：

- `__AtAutoreleasePool`
- `AutoreleasePoolPage`

真正管理 autorelease 对象的是 `AutoreleasePoolPage`。

---

# 十一、AutoreleasePoolPage 结构

## 43. AutoreleasePoolPage 是什么？

### 答案

`AutoreleasePoolPage` 是自动释放池底层用来存储 autorelease 对象地址的数据结构。

每个 `AutoreleasePoolPage` 占用 4096 字节。

它内部一部分空间用于存储成员变量，剩余空间用于存放 autorelease 对象的地址。

---

## 44. AutoreleasePoolPage 主要有哪些成员？

### 答案

典型结构中包含：

```cpp
class AutoreleasePoolPage {
    magic_t magic;
    id *next;
    pthread_t thread;
    AutoreleasePoolPage *parent;
    AutoreleasePoolPage *child;
    uint32_t depth;
    uint32_t hiwat;
};
```

字段含义：

| 字段 | 作用 |
|---|---|
| magic | 校验结构完整性 |
| next | 指向下一个可存储 autorelease 对象地址的位置 |
| thread | 当前 page 所属线程 |
| parent | 上一个 page |
| child | 下一个 page |
| depth | 当前 page 深度 |
| hiwat | 高水位标记 |

---

## 45. 多个 AutoreleasePoolPage 如何组织？

### 答案

多个 `AutoreleasePoolPage` 通过双向链表连接。

```text
page1 <-> page2 <-> page3
```

当一个 page 存满后，会创建新的 page，并通过 `child`、`parent` 连接。

---

## 46. push 和 pop 分别做什么？

### 答案

### push

调用 `push` 时，会把一个 `POOL_BOUNDARY` 入栈，并返回它的地址。

```text
POOL_BOUNDARY
obj1
obj2
obj3
```

### pop

调用 `pop` 时，传入之前 push 返回的 `POOL_BOUNDARY` 地址。

系统会从最新加入的对象开始发送 `release`，直到遇到这个边界。

```text
release obj3
release obj2
release obj1
遇到 POOL_BOUNDARY，停止
```

---

## 47. id *next 的作用是什么？

### 答案

`next` 指向下一个可以存放 autorelease 对象地址的位置。

当一个对象调用 autorelease 后，对象地址会被写入 `next` 指向的位置，然后 `next` 后移。

可以理解为：

```text
*next = obj;
next++;
```

---

# 十二、RunLoop 与 AutoreleasePool

## 48. RunLoop 和自动释放池有什么关系？

### 答案

iOS 主线程 RunLoop 会自动管理自动释放池。

系统在主线程 RunLoop 中注册了两个 Observer。

---

## 49. 主线程 RunLoop 中自动释放池的处理时机是什么？

### 答案

主线程 RunLoop 中有两个关键 Observer：

### 第一个 Observer

监听：

```text
kCFRunLoopEntry
```

作用：

```text
objc_autoreleasePoolPush()
```

也就是 RunLoop 进入时创建自动释放池。

---

### 第二个 Observer

监听：

```text
kCFRunLoopBeforeWaiting
kCFRunLoopBeforeExit
```

在即将休眠时：

```text
objc_autoreleasePoolPop()
objc_autoreleasePoolPush()
```

在即将退出时：

```text
objc_autoreleasePoolPop()
```

---

## 50. 为什么 autorelease 对象通常在当前 RunLoop 结束前释放？

### 答案

因为主线程 RunLoop 在即将进入休眠时会 pop 自动释放池。

所以很多 autorelease 对象并不是方法结束立刻释放，而是在本轮 RunLoop 即将休眠或退出时释放。

---

# 十三、局部对象出了方法是否立即释放

## 51. 方法里有局部对象，出了方法后会立即释放吗？

### 答案

不一定，要看对象的创建方式和引用关系。

---

## 情况一：普通局部变量

```objc
- (void)test {
    int a = 10;
}
```

`a` 是栈上的普通局部变量，方法结束后栈帧销毁，变量立即失效。

---

## 情况二：局部指针指向堆对象

```objc
- (void)test {
    Person *p = [[Person alloc] init];
}
```

`p` 这个指针变量在栈上，方法结束后会销毁。

但是 `Person` 对象本体在堆上。

在 ARC 下，编译器会在合适位置释放对象。

在 MRC 下，如果没有手动 release，就会内存泄漏。

---

## 情况三：autorelease 对象

```objc
- (void)test {
    Person *p = [Person person];
}
```

如果 `person` 方法返回的是 autorelease 对象，那么方法结束后不一定立即释放，而是等自动释放池 pop 时才 release。

---

## 情况四：被其他对象强引用

```objc
- (void)test {
    Person *p = [[Person alloc] init];
    self.person = p;
}
```

即使方法结束，`p` 局部变量销毁，但对象被 `self.person` 强引用，所以不会释放。

---

# 十四、高频追问总结

## 52. strong、weak、copy、assign 分别适合什么场景？

### 答案

| 修饰符 | 作用 | 常见场景 |
|---|---|---|
| strong | 强引用，持有对象 | 普通对象属性 |
| weak | 弱引用，不持有对象，自动 nil | delegate、循环引用处理 |
| copy | 拷贝对象 | NSString、block、NSArray 等 |
| assign | 直接赋值 | int、float、BOOL、NSInteger 等基本类型 |

---

## 53. block 为什么经常用 copy？

### 答案

block 一开始可能在栈上。

如果需要在作用域外继续使用，就必须 copy 到堆上。

因此 block 属性通常写成：

```objc
@property (nonatomic, copy) void(^completion)(void);
```

虽然 ARC 下很多情况下会自动 copy，但属性语义仍建议使用 copy。

---

## 54. 什么是循环引用？

### 答案

两个或多个对象互相强引用，导致引用计数无法归零，就是循环引用。

示例：

```objc
self.block = ^{
    NSLog(@"%@", self);
};
```

引用关系：

```text
self -> block
block -> self
```

解决：

```objc
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    NSLog(@"%@", strongSelf);
};
```

---

## 55. 为什么 block 内通常 weak-strong dance？

### 答案

直接使用 weakSelf：

```objc
__weak typeof(self) weakSelf = self;
self.block = ^{
    [weakSelf doSomething];
};
```

问题是 block 执行过程中，weakSelf 可能突然变成 nil。

所以常见写法：

```objc
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf doSomething];
};
```

这样可以保证 block 执行期间对象临时被强引用，避免执行中途释放。

---

## 56. autoreleasepool 在实际开发中有什么用途？

### 答案

常用于大量临时对象场景，避免内存峰值过高。

例如：

```objc
for (int i = 0; i < 100000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"item-%d", i];
        NSLog(@"%@", str);
    }
}
```

如果不加 `@autoreleasepool`，大量 autorelease 对象可能堆积到当前 RunLoop 结束才释放，造成内存峰值过高。

---

# 十五、综合代码题

## 57. 下面代码是否会循环引用？

```objc
@interface ViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1
                                                  target:self
                                                selector:@selector(test)
                                                userInfo:nil
                                                 repeats:YES];
}

- (void)test {
    NSLog(@"test");
}

@end
```

### 答案

会。

因为：

```text
ViewController 强引用 timer
timer 强引用 target，也就是 ViewController
```

形成：

```text
ViewController -> NSTimer -> ViewController
```

导致控制器无法释放。

解决方式：

1. 使用 block + weak self
2. 使用 NSProxy
3. 在合适时机 invalidate
4. 使用 GCD Timer

---

## 58. 下面代码输出什么？

```objc
NSMutableString *str1 = [NSMutableString stringWithString:@"abc"];
NSString *str2 = [str1 copy];
[str1 appendString:@"123"];
NSLog(@"%@", str1);
NSLog(@"%@", str2);
```

### 答案

输出：

```text
abc123
abc
```

原因：

- `str1` 是可变字符串。
- `[str1 copy]` 会产生一个不可变副本。
- 修改 `str1` 不会影响 `str2`。

---

## 59. 下面属性为什么 name 要用 copy？

```objc
@property (nonatomic, copy) NSString *name;
```

### 答案

为了防止外部传入可变字符串后继续修改。

例如：

```objc
NSMutableString *str = [NSMutableString stringWithString:@"Tom"];
person.name = str;
[str appendString:@" Jack"];
```

如果属性是 strong，`person.name` 可能也会被改掉。

如果属性是 copy，赋值时生成不可变副本，外部修改不会影响对象内部状态。

---

## 60. 下面代码中对象什么时候释放？

```objc
- (void)test {
    NSString *str = [NSString stringWithFormat:@"hello"];
    NSLog(@"%@", str);
}
```

### 答案

`stringWithFormat:` 通常返回 autorelease 对象。

所以它一般不会在方法结束时立即释放，而是会在当前自动释放池 pop 时收到 release。

在主线程中，通常是在本轮 RunLoop 即将休眠或退出时释放。

不过 ARC 和编译器优化可能改变具体释放点，但核心理解是：autorelease 对象延迟释放。

---

## 61. 下面代码是否需要加 autoreleasepool？

```objc
for (int i = 0; i < 100000; i++) {
    NSString *str = [NSString stringWithFormat:@"%d", i];
    NSLog(@"%@", str);
}
```

### 答案

建议加。

因为循环中会创建大量临时 autorelease 对象，如果等到当前 RunLoop 结束再统一释放，可能造成内存峰值过高。

优化：

```objc
for (int i = 0; i < 100000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"%d", i];
        NSLog(@"%@", str);
    }
}
```

这样每次循环结束都会释放临时对象，降低内存峰值。

---

## 62. weak 指针为什么不会野指针？

### 答案

因为 Runtime 会维护弱引用表。

对象销毁时，Runtime 会找到所有指向该对象的 weak 指针，并把它们设置为 nil。

所以访问 weak 指针时，如果对象已释放，拿到的是 nil，而不是悬空地址。

---

## 63. assign 修饰对象会有什么问题？

### 答案

`assign` 不会强引用对象，也不会在对象释放后自动置 nil。

如果对象释放后继续访问 assign 指针，就可能访问野指针并崩溃。

错误示例：

```objc
@property (nonatomic, assign) NSObject *obj;
```

对象类型一般不要用 assign，应该使用 weak 或 strong。

---

## 64. autoreleasepool 的 push/pop 机制如何描述？

### 答案

可以这样描述：

1. `push` 时压入一个 `POOL_BOUNDARY`。
2. autorelease 对象依次压入 AutoreleasePoolPage。
3. `pop` 时从最新对象开始 release。
4. 一直释放到遇到 `POOL_BOUNDARY` 为止。

示意：

```text
push -> POOL_BOUNDARY
autorelease obj1
autorelease obj2
autorelease obj3
pop -> release obj3, obj2, obj1
```

---

## 65. 自动释放池和线程有什么关系？

### 答案

自动释放池是和线程相关的。

每个线程都有自己的 autorelease pool 栈。

主线程系统会自动创建和管理自动释放池。

子线程如果创建大量 autorelease 对象，建议手动添加：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    @autoreleasepool {
        // 创建临时对象
    }
});
```

---

# 十六、最终复习重点

## 必背知识点

1. `NSTimer`、`CADisplayLink` 会强引用 target，容易循环引用。
2. `NSTimer` 依赖 RunLoop，不一定准时。
3. GCD Timer 不依赖 RunLoop，通常更准。
4. iOS 内存布局：代码段、数据段、堆、栈、内核区。
5. OC 对象本体一般在堆上，局部指针变量在栈上。
6. Tagged Pointer 把小对象数据直接存储在指针中。
7. ARC 是编译器自动插入 retain/release，不是 GC。
8. ARC 不能自动解决循环引用。
9. weak 不增加引用计数，对象释放后自动置 nil。
10. weak 底层依赖 SideTable 和 weak_table。
11. copy 返回不可变对象，mutableCopy 返回可变对象。
12. 不可变对象 copy 通常浅拷贝，可变对象 copy 通常深拷贝。
13. 引用计数可能存在 isa 中，也可能存在 SideTable 中。
14. dealloc 调用链：dealloc -> _objc_rootDealloc -> rootDealloc -> object_dispose -> objc_destructInstance -> free。
15. autorelease 对象在自动释放池 pop 时 release。
16. AutoreleasePoolPage 每页 4096 字节，双向链表连接。
17. RunLoop Entry 时 push，BeforeWaiting 时 pop + push，BeforeExit 时 pop。
18. 方法中的局部对象出了方法不一定立即释放，要看对象类型、引用关系和 autorelease 情况。

---

# 十七、一句话总结

iOS 内存管理的核心是引用计数。ARC 只是帮我们自动插入引用计数管理代码，但对象之间的强引用关系仍然需要开发者理解和设计。真正要掌握内存管理，需要理解对象存储区域、引用计数位置、weak 表、自动释放池、RunLoop 释放时机以及常见循环引用场景。
