# iOS 多线程题库与详细答案

> 来源：用户上传的《04-多线程.pptx》整理。内容覆盖 iOS 多线程方案、GCD、队列、线程安全、锁、读写安全、atomic、代码打印结果等核心知识点。

---

## 目录

1. 多线程基础
2. iOS 常见多线程方案
3. GCD 核心概念
4. 同步、异步、串行、并发
5. GCD 执行效果与死锁
6. 代码打印结果分析
7. GCD 常用能力
8. GCD 与 NSOperationQueue 对比
9. 线程安全问题
10. iOS 常见锁
11. 自旋锁与互斥锁
12. 手写锁代码
13. atomic 分析
14. 读写安全
15. 项目实战题
16. 高频总结

---

# 一、多线程基础

## 1. 你理解的多线程是什么？

多线程就是在一个进程中同时存在多条执行路径，每条执行路径称为线程。

在 iOS 中，一个 App 本质上是一个进程。进程启动后默认会创建一条主线程，主线程主要负责 UI 渲染、事件响应、RunLoop 运行等任务。除了主线程之外，我们还可以创建子线程，把耗时任务放到子线程中执行，例如网络请求、图片解码、文件读写、数据库操作、复杂计算等。

多线程的核心价值是：

1. 提升程序响应速度
2. 避免主线程卡顿
3. 充分利用多核 CPU
4. 将耗时任务和 UI 任务分离

但是多线程也会带来问题，比如：

1. 线程安全问题
2. 资源竞争问题
3. 死锁问题
4. 线程切换开销
5. 调试难度增加

所以多线程不是越多越好，而是要合理使用。

---

## 2. iOS 中常见的多线程方案有哪些？

iOS 中常见的多线程方案有四种：

| 方案 | 语言 | 特点 | 使用频率 |
|---|---|---|---|
| pthread | C | 跨平台、底层、使用复杂 | 几乎不用 |
| NSThread | OC | 面向对象，可直接操作线程对象 | 偶尔使用 |
| GCD | C | 自动管理线程，充分利用多核 | 经常使用 |
| NSOperation / NSOperationQueue | OC | 基于 GCD，面向对象，功能更丰富 | 经常使用 |

### pthread

pthread 是 POSIX 线程，属于 C 语言接口，跨平台，Linux、Unix、iOS 都可以使用。优点是底层、灵活、可移植；缺点是使用复杂，需要开发者手动管理线程生命周期。

### NSThread

NSThread 是 Objective-C 对线程的封装，可以直接创建和操作线程对象。它比 pthread 简单，但仍然需要开发者自己管理线程执行逻辑。

### GCD

GCD，全称 Grand Central Dispatch，是 Apple 提供的一套多线程解决方案。它不要求开发者直接管理线程，而是通过“队列 + 任务”的方式执行代码。GCD 会根据系统资源自动创建、复用和调度线程。

### NSOperation / NSOperationQueue

NSOperation 是对任务的封装，NSOperationQueue 是任务队列。它底层基于 GCD，但提供了更多高级功能，比如取消任务、设置依赖关系、设置最大并发数、监听任务状态等。

---

## 3. 你更倾向于使用哪一种多线程方案？为什么？

日常开发中，更常用的是 GCD 和 NSOperationQueue。

如果只是简单异步任务，比如网络请求后回到主线程刷新 UI，通常使用 GCD，因为语法简单，性能好，使用方便。

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    // 子线程执行耗时任务
    NSData *data = [NSData dataWithContentsOfURL:url];

    dispatch_async(dispatch_get_main_queue(), ^{
        // 主线程刷新 UI
        self.imageView.image = [UIImage imageWithData:data];
    });
});
```

如果任务之间有依赖关系，或者需要取消、暂停、设置最大并发数，可以使用 NSOperationQueue。

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 3;

NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"下载图片");
}];

NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"解码图片");
}];

[op2 addDependency:op1];

[queue addOperation:op1];
[queue addOperation:op2];
```

总结：

简单任务用 GCD。
复杂任务管理用 NSOperationQueue。

---

# 二、GCD 核心概念

## 4. GCD 是什么？

GCD 是 Grand Central Dispatch 的缩写，是 Apple 提供的多线程解决方案。

GCD 的核心思想是：开发者只需要把任务添加到队列中，系统会自动负责线程的创建、调度、复用和销毁。

GCD 中有两个核心概念：

1. 任务：要执行的代码块，也就是 block
2. 队列：用来存放任务，决定任务的执行顺序

GCD 的优势：

1. 使用简单
2. 自动管理线程生命周期
3. 能充分利用多核 CPU
4. 性能较好
5. 可以很方便地实现异步、同步、延迟、一次性执行、队列组、栅栏函数等功能

---

## 5. GCD 中同步和异步的区别是什么？

GCD 中执行任务常用两个函数：

```objc
dispatch_sync(queue, block);
dispatch_async(queue, block);
```

### dispatch_sync

同步执行任务。

特点：

1. 不具备开启新线程的能力
2. 当前任务执行完毕后，才会继续向下执行
3. 会阻塞当前线程
4. 任务通常在当前线程执行

示例：

```objc
NSLog(@"1");

dispatch_sync(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
});

NSLog(@"3");
```

打印结果：

```objc
1
2
3
```

因为 sync 会等待 block 执行完毕后，才继续执行后面的代码。

### dispatch_async

异步执行任务。

特点：

1. 具备开启新线程的能力
2. 不会阻塞当前线程
3. 添加完任务后立即返回
4. 任务什么时候执行由系统调度决定

示例：

```objc
NSLog(@"1");

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
});

NSLog(@"3");
```

常见打印结果：

```objc
1
3
2
```

因为 async 不会等待 block 执行完成。

---

## 6. 同步、异步、串行、并发分别影响什么？

这四个概念很容易混淆。

### 同步和异步影响：是否具备开启新线程的能力

同步：不具备开启新线程的能力，一般在当前线程执行任务。
异步：具备开启新线程的能力，不会阻塞当前线程。

### 串行和并发影响：任务的执行方式

串行：任务一个接一个执行。
并发：多个任务可以同时执行。

| 概念 | 影响点 | 说明 |
|---|---|---|
| 同步 | 是否阻塞当前线程 | 会阻塞 |
| 异步 | 是否阻塞当前线程 | 不会阻塞 |
| 串行 | 任务执行顺序 | 一个接一个 |
| 并发 | 任务执行顺序 | 多个任务同时执行 |

---

## 7. GCD 的队列有哪些类型？

GCD 队列主要分为两大类：

1. 串行队列
2. 并发队列

另外从来源上还可以分为：

1. 主队列
2. 全局并发队列
3. 自定义队列

### 串行队列

串行队列中的任务一个接一个执行。前一个任务执行完，后一个任务才会开始。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.serial", DISPATCH_QUEUE_SERIAL);
```

### 并发队列

并发队列允许多个任务同时执行。但并发能力只有在使用 `dispatch_async` 时才有效。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.concurrent", DISPATCH_QUEUE_CONCURRENT);
```

### 主队列

主队列是一个特殊的串行队列，专门在主线程执行任务。

```objc
dispatch_queue_t mainQueue = dispatch_get_main_queue();
```

主队列常用于刷新 UI：

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    self.label.text = @"更新 UI";
});
```

### 全局并发队列

全局并发队列由系统提供，是并发队列。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

常用于执行耗时任务：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"耗时任务");
});
```

---

# 三、GCD 执行效果与死锁

## 8. 同步函数 + 串行队列，会不会开启新线程？

不会。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.serial", DISPATCH_QUEUE_SERIAL);

dispatch_sync(queue, ^{
    NSLog(@"任务1 %@", [NSThread currentThread]);
});

dispatch_sync(queue, ^{
    NSLog(@"任务2 %@", [NSThread currentThread]);
});
```

结果：

1. 不会开启新线程
2. 任务串行执行
3. 在当前线程执行

因为 `dispatch_sync` 不具备开启新线程的能力，而串行队列本身要求任务一个接一个执行。

---

## 9. 异步函数 + 串行队列，会不会开启新线程？

会开启新线程，但任务仍然串行执行。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.serial", DISPATCH_QUEUE_SERIAL);

dispatch_async(queue, ^{
    NSLog(@"任务1 %@", [NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"任务2 %@", [NSThread currentThread]);
});
```

特点：

1. 会开启新线程
2. 两个任务按顺序执行
3. 不会并发执行
4. 一般只开一条子线程执行这个串行队列中的任务

---

## 10. 同步函数 + 并发队列，会不会开启新线程？

不会。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.concurrent", DISPATCH_QUEUE_CONCURRENT);

dispatch_sync(queue, ^{
    NSLog(@"任务1 %@", [NSThread currentThread]);
});

dispatch_sync(queue, ^{
    NSLog(@"任务2 %@", [NSThread currentThread]);
});
```

结果：

1. 不会开启新线程
2. 任务串行执行
3. 在当前线程执行

原因是同步函数不具备开启新线程的能力。即使队列是并发队列，也无法并发起来。

---

## 11. 异步函数 + 并发队列，会不会开启新线程？

会。

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.concurrent", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, ^{
    NSLog(@"任务1 %@", [NSThread currentThread]);
});

dispatch_async(queue, ^{
    NSLog(@"任务2 %@", [NSThread currentThread]);
});
```

特点：

1. 会开启新线程
2. 多个任务可以并发执行
3. 任务完成顺序不确定
4. 是 GCD 中最常见的异步并发场景

---

## 12. 异步函数 + 主队列，会不会开启新线程？

不会。

主队列中的任务只会在主线程执行。

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"任务 %@", [NSThread currentThread]);
});
```

结果：

1. 不会开启新线程
2. 任务在主线程执行
3. 主队列是串行队列，所以任务一个接一个执行

---

## 13. 同步函数 + 主队列，会发生什么？

如果在主线程中执行下面代码，会死锁：

```objc
NSLog(@"1");

dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"2");
});

NSLog(@"3");
```

打印结果：

```objc
1
```

然后卡死。

原因：

1. `dispatch_sync` 会阻塞当前线程，等待 block 执行完毕
2. 当前线程是主线程
3. block 被添加到主队列
4. 主队列中的任务必须在主线程执行
5. 但是主线程正在等待 block 执行完
6. block 又在等待主线程空闲后执行
7. 双方互相等待，形成死锁

本质是：在当前串行队列中同步添加任务到当前队列，会死锁。

---

## 14. 使用 sync 往当前串行队列添加任务为什么会死锁？

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.serial", DISPATCH_QUEUE_SERIAL);

dispatch_async(queue, ^{
    NSLog(@"1");

    dispatch_sync(queue, ^{
        NSLog(@"2");
    });

    NSLog(@"3");
});
```

打印结果：

```objc
1
```

然后死锁。

原因：

外层任务已经在 `queue` 这个串行队列中执行。串行队列一次只能执行一个任务。外层任务执行到 `dispatch_sync` 时，会等待内层任务执行完毕。但是内层任务也被添加到了同一个串行队列中，必须等外层任务执行完才能执行。

于是：

外层任务等待内层任务完成。
内层任务等待外层任务结束。
最终死锁。

---

# 四、代码打印结果分析

## 15. 下面代码打印结果是什么？

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"1");

    dispatch_queue_t queue = dispatch_queue_create("com.demo.serial", DISPATCH_QUEUE_SERIAL);

    dispatch_async(queue, ^{
        NSLog(@"2");

        [self performSelector:@selector(test) withObject:nil afterDelay:0];

        NSLog(@"3");
    });

    NSLog(@"4");
}

- (void)test {
    NSLog(@"5");
}
```

常见打印结果：

```objc
1
4
2
3
```

不会打印 `5`。

原因：

`performSelector:withObject:afterDelay:` 的本质是往当前线程的 RunLoop 中添加定时器。

但是 GCD 创建的子线程默认没有启动 RunLoop，所以这个定时器不会被触发，`test` 方法不会执行。

如果想让它执行，需要让当前子线程的 RunLoop 跑起来，例如：

```objc
dispatch_async(queue, ^{
    NSLog(@"2");

    [self performSelector:@selector(test) withObject:nil afterDelay:0];

    [[NSRunLoop currentRunLoop] run];

    NSLog(@"3");
});
```

不过这样会导致 RunLoop 一直运行，`NSLog(@"3")` 可能不会马上执行。

---

## 16. 为什么 performSelector:afterDelay: 在子线程中可能不执行？

因为它依赖 RunLoop。

`performSelector:withObject:afterDelay:` 底层会创建一个定时器，并把定时器添加到当前线程的 RunLoop 中。

主线程默认开启了 RunLoop，所以主线程中可以正常执行。

但子线程默认没有开启 RunLoop。如果在子线程调用 afterDelay 方法，定时器虽然被添加了，但是 RunLoop 不运行，定时器就没有机会触发。

所以不会执行目标方法。

---

## 17. 如何让子线程中的 performSelector:afterDelay: 生效？

可以手动启动当前线程的 RunLoop：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [self performSelector:@selector(test) withObject:nil afterDelay:1];

    [[NSRunLoop currentRunLoop] run];
});
```

或者使用 GCD 的延迟函数替代：

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)),
               dispatch_get_main_queue(), ^{
    [self test];
});
```

在实际开发中，更推荐使用 `dispatch_after`，因为更简单直接。

---

# 五、GCD 常用能力

## 18. GCD 如何实现延迟执行？

使用 `dispatch_after`。

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)),
               dispatch_get_main_queue(), ^{
    NSLog(@"2 秒后执行");
});
```

注意：

`dispatch_after` 不是严格意义上的“2 秒后立即执行”，而是 2 秒后把任务提交到指定队列。任务真正执行的时间还取决于队列当前是否空闲。

---

## 19. GCD 如何实现一次性执行？

使用 `dispatch_once`。

```objc
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    NSLog(@"只执行一次");
});
```

它常用于单例：

```objc
+ (instancetype)sharedInstance {
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}
```

特点：

1. 线程安全
2. 整个 App 生命周期内只执行一次
3. 常用于单例初始化

---

## 20. GCD 如何实现队列组？

使用 `dispatch_group_t`。

需求：任务 1 和任务 2 异步并发执行，等两个任务都完成后，回到主线程执行任务 3。

```objc
dispatch_group_t group = dispatch_group_create();
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

dispatch_group_async(group, queue, ^{
    NSLog(@"任务1 %@", [NSThread currentThread]);
});

dispatch_group_async(group, queue, ^{
    NSLog(@"任务2 %@", [NSThread currentThread]);
});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"任务3 %@", [NSThread currentThread]);
});
```

执行逻辑：

1. 任务 1 和任务 2 加入同一个 group
2. 两个任务在全局并发队列中异步执行
3. group 中的任务全部执行完后
4. `dispatch_group_notify` 回到主队列执行任务 3

---

## 21. dispatch_group_enter 和 dispatch_group_leave 用在什么场景？

当任务不是直接通过 `dispatch_group_async` 添加时，比如任务内部还有异步网络请求，就需要手动使用 `enter` 和 `leave`。

```objc
dispatch_group_t group = dispatch_group_create();

dispatch_group_enter(group);
[self requestAWithCompletion:^{
    NSLog(@"请求 A 完成");
    dispatch_group_leave(group);
}];

dispatch_group_enter(group);
[self requestBWithCompletion:^{
    NSLog(@"请求 B 完成");
    dispatch_group_leave(group);
}];

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"所有请求完成，刷新 UI");
});
```

注意：

`dispatch_group_enter` 和 `dispatch_group_leave` 必须成对出现。
如果 enter 多于 leave，group 永远不会完成。
如果 leave 多于 enter，可能会崩溃。

---

## 22. GCD 如何控制最大并发数？

可以使用 `dispatch_semaphore`。

例如最多允许 3 个任务同时执行：

```objc
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
dispatch_semaphore_t semaphore = dispatch_semaphore_create(3);

for (NSInteger i = 0; i < 10; i++) {
    dispatch_async(queue, ^{
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

        NSLog(@"执行任务 %ld %@", (long)i, [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2];

        dispatch_semaphore_signal(semaphore);
    });
}
```

逻辑：

1. 信号量初始值为 3
2. 每个任务开始前 wait，信号量减 1
3. 如果信号量为 0，后续任务等待
4. 任务结束后 signal，信号量加 1
5. 同一时间最多 3 个任务执行

---

# 六、GCD 与 NSOperationQueue 对比

## 23. NSOperationQueue 和 GCD 有什么区别？

GCD 是 C 语言 API，偏底层，核心是队列和 block。
NSOperationQueue 是 OC API，底层基于 GCD，使用更加面向对象。

### GCD 优点

1. 简单轻量
2. 性能好
3. 适合简单异步任务
4. 适合回到主线程刷新 UI
5. 适合使用队列组、信号量、栅栏函数

### NSOperationQueue 优点

1. 可以取消任务
2. 可以设置任务依赖
3. 可以设置最大并发数
4. 可以监听任务状态
5. 可以暂停和恢复队列
6. 面向对象，适合复杂任务管理

| 对比点 | GCD | NSOperationQueue |
|---|---|---|
| API 类型 | C | OC |
| 使用方式 | block + queue | operation + queue |
| 是否可取消 | 不方便 | 支持 |
| 是否支持依赖 | 不直接支持 | 支持 |
| 最大并发数 | 需自己控制 | 直接支持 |
| 任务状态 | 不方便监听 | 支持 |
| 适合场景 | 简单任务 | 复杂任务 |

---

## 24. NSOperationQueue 如何设置最大并发数？

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 3;
```

含义是同一时间最多执行 3 个任务。

如果设置为 1：

```objc
queue.maxConcurrentOperationCount = 1;
```

那么这个队列就相当于串行队列。

---

## 25. NSOperationQueue 如何设置依赖关系？

```objc
NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"下载图片");
}];

NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"解码图片");
}];

NSBlockOperation *op3 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"刷新 UI");
}];

[op2 addDependency:op1];
[op3 addDependency:op2];

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperations:@[op1, op2, op3] waitUntilFinished:NO];
```

执行顺序：

```objc
op1 -> op2 -> op3
```

---

# 七、线程安全问题

## 26. 什么是线程安全问题？

当多个线程同时访问同一份资源时，如果没有做好同步控制，就可能出现数据错乱，这就是线程安全问题。

共享资源可以是：

1. 同一个对象
2. 同一个变量
3. 同一个文件
4. 同一个数据库
5. 同一个数组或字典

例如多个线程同时卖票：

```objc
self.ticketCount = 1000;
```

两个线程同时读取到 1000，然后都执行减 1：

线程 A：1000 - 1 = 999
线程 B：1000 - 1 = 999

最终票数还是 999，而不是 998。

这就是典型的数据错乱。

---

## 27. 如何解决线程安全问题？

核心方案是：线程同步。

也就是让多个线程按照规定的顺序访问共享资源。常见方式是加锁。

常见方案包括：

1. os_unfair_lock
2. pthread_mutex
3. dispatch_semaphore
4. 串行队列
5. NSLock
6. NSRecursiveLock
7. NSCondition
8. NSConditionLock
9. @synchronized
10. pthread_rwlock
11. dispatch_barrier_async

---

## 28. 加锁的本质是什么？

加锁的本质是保护临界区。

临界区就是访问共享资源的代码区域。

```objc
- (void)saleTicket {
    [self.lock lock];

    NSInteger count = self.ticketCount;
    count--;
    self.ticketCount = count;

    [self.lock unlock];
}
```

加锁后，同一时间只允许一条线程进入临界区，其他线程必须等待。

这样可以保证共享资源不会被多个线程同时修改。

---

# 八、iOS 常见锁

## 29. iOS 中常见的锁有哪些？

常见锁包括：

1. OSSpinLock
2. os_unfair_lock
3. pthread_mutex
4. dispatch_semaphore
5. 串行队列
6. NSLock
7. NSRecursiveLock
8. NSCondition
9. NSConditionLock
10. @synchronized
11. pthread_rwlock

---

## 30. OSSpinLock 是什么？为什么不安全？

OSSpinLock 是自旋锁。

特点是：等待锁的线程不会休眠，而是一直循环等待，也就是 busy-wait。

伪代码类似：

```objc
while (lockIsBusy) {
    // 一直循环等待
}
```

问题是它可能出现优先级反转。

假设：

1. 低优先级线程拿到了锁
2. 高优先级线程尝试加锁
3. 高优先级线程一直占用 CPU 自旋等待
4. 低优先级线程因为抢不到 CPU，无法执行完并释放锁
5. 高优先级线程也一直拿不到锁

最终导致严重卡顿。

所以 OSSpinLock 已经不推荐使用。

---

## 31. os_unfair_lock 是什么？

`os_unfair_lock` 是用来替代 OSSpinLock 的锁，从 iOS 10 开始支持。

它不是忙等锁。等待锁的线程会进入休眠状态，而不是一直占用 CPU。

```objc
#import <os/lock.h>

@property (nonatomic, assign) os_unfair_lock lock;
```

初始化：

```objc
_lock = OS_UNFAIR_LOCK_INIT;
```

加锁解锁：

```objc
os_unfair_lock_lock(&_lock);

// 临界区
self.money += 100;

os_unfair_lock_unlock(&_lock);
```

特点：

1. 性能高
2. 替代 OSSpinLock
3. 等待线程会休眠
4. 不是递归锁
5. 不要重复加同一把锁，否则可能死锁

---

## 32. pthread_mutex 是什么？

`pthread_mutex` 是互斥锁。

等待锁的线程会进入休眠状态，不会一直占用 CPU。

```objc
#import <pthread.h>

@property (nonatomic, assign) pthread_mutex_t mutex;
```

初始化：

```objc
pthread_mutex_init(&_mutex, NULL);
```

加锁解锁：

```objc
pthread_mutex_lock(&_mutex);

// 临界区
self.money += 100;

pthread_mutex_unlock(&_mutex);
```

销毁：

```objc
pthread_mutex_destroy(&_mutex);
```

---

## 33. pthread_mutex 如何实现递归锁？

普通 mutex 不能递归加锁。如果同一线程重复加锁，会死锁。

递归锁允许同一线程对同一把锁重复加锁。

```objc
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);

pthread_mutex_init(&_mutex, &attr);

pthread_mutexattr_destroy(&attr);
```

使用：

```objc
- (void)test {
    pthread_mutex_lock(&_mutex);

    NSLog(@"test");

    [self test2];

    pthread_mutex_unlock(&_mutex);
}

- (void)test2 {
    pthread_mutex_lock(&_mutex);

    NSLog(@"test2");

    pthread_mutex_unlock(&_mutex);
}
```

如果是普通锁，上面代码会死锁。
如果是递归锁，就不会。

---

## 34. NSLock 是什么？

NSLock 是对 pthread_mutex 普通锁的封装。

```objc
@property (nonatomic, strong) NSLock *lock;
```

初始化：

```objc
self.lock = [[NSLock alloc] init];
```

加锁解锁：

```objc
[self.lock lock];

// 临界区
self.ticketCount--;

[self.lock unlock];
```

优点是使用简单。缺点是性能不如底层锁。

---

## 35. NSRecursiveLock 是什么？

NSRecursiveLock 是递归锁，允许同一线程重复加锁。

适合递归调用或者一个加锁方法内部调用另一个也会加锁的方法。

```objc
@property (nonatomic, strong) NSRecursiveLock *lock;

- (void)test {
    [self.lock lock];

    NSLog(@"test");

    [self test2];

    [self.lock unlock];
}

- (void)test2 {
    [self.lock lock];

    NSLog(@"test2");

    [self.lock unlock];
}
```

如果用 NSLock，上面代码可能死锁。
如果用 NSRecursiveLock，则可以正常执行。

---

## 36. NSCondition 是什么？

NSCondition 是对 mutex 和 condition 的封装。

它不仅能加锁，还能让线程等待某个条件，或者唤醒等待的线程。

常见 API：

```objc
- lock
- unlock
- wait
- signal
- broadcast
```

示例：

```objc
@property (nonatomic, strong) NSCondition *condition;
@property (nonatomic, assign) BOOL hasData;

- (void)consumer {
    [self.condition lock];

    while (!self.hasData) {
        [self.condition wait];
    }

    NSLog(@"消费数据");

    [self.condition unlock];
}

- (void)producer {
    [self.condition lock];

    self.hasData = YES;
    NSLog(@"生产数据");

    [self.condition signal];

    [self.condition unlock];
}
```

注意这里建议使用 `while` 判断条件，而不是 `if`，因为线程被唤醒后条件不一定仍然成立。

---

## 37. NSConditionLock 是什么？

NSConditionLock 是对 NSCondition 的进一步封装，可以给锁设置具体条件值。

```objc
NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:1];

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [lock lockWhenCondition:1];

    NSLog(@"任务1");

    [lock unlockWithCondition:2];
});

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [lock lockWhenCondition:2];

    NSLog(@"任务2");

    [lock unlock];
});
```

执行顺序：

```objc
任务1
任务2
```

因为任务 2 必须等条件变成 2 才能执行。

---

## 38. dispatch_semaphore 如何保证线程安全？

当信号量初始值为 1 时，它可以当成锁来使用。

```objc
@property (nonatomic, strong) dispatch_semaphore_t semaphore;
```

初始化：

```objc
self.semaphore = dispatch_semaphore_create(1);
```

使用：

```objc
dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);

// 临界区
self.money += 100;

dispatch_semaphore_signal(self.semaphore);
```

逻辑：

1. 初始值为 1
2. 一个线程进入时，wait 后变成 0
3. 其他线程再 wait 时会等待
4. 当前线程执行完 signal 后变成 1
5. 其他线程才能进入

---

## 39. 串行队列如何保证线程安全？

因为串行队列一次只执行一个任务，所以可以把所有读写共享资源的操作都放到同一个串行队列中。

```objc
@property (nonatomic, strong) dispatch_queue_t queue;
```

初始化：

```objc
self.queue = dispatch_queue_create("com.demo.safe", DISPATCH_QUEUE_SERIAL);
```

写数据：

```objc
dispatch_async(self.queue, ^{
    self.money += 100;
});
```

读数据：

```objc
dispatch_sync(self.queue, ^{
    NSLog(@"money = %ld", self.money);
});
```

这种方式本质上是利用串行队列保证同一时间只有一个任务访问共享资源。

---

## 40. @synchronized 是什么？

`@synchronized` 是 Objective-C 提供的同步锁。

```objc
@synchronized (self) {
    self.money += 100;
}
```

它内部会根据传入的对象生成对应的递归锁，然后进行加锁和解锁。

特点：

1. 使用简单
2. 自动加锁和解锁
3. 是递归锁
4. 性能较差
5. 不适合高频调用的性能敏感场景

---

# 九、自旋锁与互斥锁

## 41. 自旋锁和互斥锁有什么区别？

### 自旋锁

线程拿不到锁时，不会休眠，而是一直循环等待。

特点：

1. 不会发生线程睡眠和唤醒的上下文切换
2. 等待时间很短时性能较好
3. 会一直占用 CPU
4. 等待时间长时浪费 CPU
5. 可能出现优先级反转问题

### 互斥锁

线程拿不到锁时，会进入休眠状态。

特点：

1. 不会一直占用 CPU
2. 适合等待时间较长的场景
3. 存在线程睡眠和唤醒开销
4. 临界区复杂时更合适

---

## 42. 什么情况下使用自旋锁比较划算？

适合以下场景：

1. 预计等待锁的时间很短
2. 临界区代码非常简单
3. 加锁操作很频繁，但锁竞争很少
4. CPU 资源不紧张
5. 多核处理器环境

例如只是修改一个简单变量，且竞争概率很低，自旋锁可能比较合适。

不过在 iOS 中不推荐再使用 OSSpinLock，因为它已经不安全。

---

## 43. 什么情况下使用互斥锁比较划算？

适合以下场景：

1. 预计等待锁的时间较长
2. 临界区有 IO 操作
3. 临界区代码复杂
4. 临界区有循环或耗时计算
5. 锁竞争激烈
6. 单核处理器

例如文件读写、数据库操作、网络相关状态更新等，更适合互斥锁。

---

## 44. 使用锁需要注意什么？

使用锁时要注意：

1. 加锁和解锁必须成对出现
2. 尽量缩小加锁范围
3. 不要在锁里面执行耗时操作
4. 不要在锁里面发起同步网络请求
5. 不要在锁里面调用复杂外部方法
6. 避免多个锁交叉使用导致死锁
7. 递归调用场景要使用递归锁
8. 锁对象要稳定，不能随意变化
9. 注意异常路径也要解锁
10. 优先选择合适的锁，而不是盲目使用重锁

错误示例：

```objc
[self.lock lock];

[self doSomething];

if (error) {
    return;
}

[self.lock unlock];
```

这里如果提前 return，就不会解锁，导致死锁。

更安全的写法：

```objc
[self.lock lock];

if (error) {
    [self.lock unlock];
    return;
}

[self doSomething];

[self.lock unlock];
```

---

# 十、手写锁代码

## 45. 用 C 实现一个互斥锁保护卖票逻辑

```objc
#import <pthread.h>

@interface TicketManager : NSObject

@property (nonatomic, assign) NSInteger ticketCount;
@property (nonatomic, assign) pthread_mutex_t mutex;

@end

@implementation TicketManager

- (instancetype)init {
    self = [super init];
    if (self) {
        self.ticketCount = 100;
        pthread_mutex_init(&_mutex, NULL);
    }
    return self;
}

- (void)saleTicket {
    pthread_mutex_lock(&_mutex);

    if (self.ticketCount > 0) {
        self.ticketCount--;
        NSLog(@"剩余票数：%ld %@", (long)self.ticketCount, [NSThread currentThread]);
    } else {
        NSLog(@"票已经卖完了");
    }

    pthread_mutex_unlock(&_mutex);
}

- (void)dealloc {
    pthread_mutex_destroy(&_mutex);
}

@end
```

使用：

```objc
TicketManager *manager = [[TicketManager alloc] init];

dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

for (NSInteger i = 0; i < 5; i++) {
    dispatch_async(queue, ^{
        for (NSInteger j = 0; j < 30; j++) {
            [manager saleTicket];
        }
    });
}
```

这个实现通过 `pthread_mutex` 保证同一时间只有一个线程可以修改 `ticketCount`。

---

## 46. 用 os_unfair_lock 实现线程安全卖票

```objc
#import <os/lock.h>

@interface TicketManager : NSObject

@property (nonatomic, assign) NSInteger ticketCount;
@property (nonatomic, assign) os_unfair_lock lock;

@end

@implementation TicketManager

- (instancetype)init {
    self = [super init];
    if (self) {
        self.ticketCount = 100;
        self.lock = OS_UNFAIR_LOCK_INIT;
    }
    return self;
}

- (void)saleTicket {
    os_unfair_lock_lock(&_lock);

    if (self.ticketCount > 0) {
        self.ticketCount--;
        NSLog(@"剩余票数：%ld %@", (long)self.ticketCount, [NSThread currentThread]);
    } else {
        NSLog(@"票已经卖完");
    }

    os_unfair_lock_unlock(&_lock);
}

@end
```

注意：

`os_unfair_lock` 不是递归锁，不能在同一线程中重复加锁。

---

## 47. 用 dispatch_semaphore 实现线程安全卖票

```objc
@interface TicketManager : NSObject

@property (nonatomic, assign) NSInteger ticketCount;
@property (nonatomic, strong) dispatch_semaphore_t semaphore;

@end

@implementation TicketManager

- (instancetype)init {
    self = [super init];
    if (self) {
        self.ticketCount = 100;
        self.semaphore = dispatch_semaphore_create(1);
    }
    return self;
}

- (void)saleTicket {
    dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);

    if (self.ticketCount > 0) {
        self.ticketCount--;
        NSLog(@"剩余票数：%ld %@", (long)self.ticketCount, [NSThread currentThread]);
    } else {
        NSLog(@"票已经卖完");
    }

    dispatch_semaphore_signal(self.semaphore);
}

@end
```

当信号量初始值为 1 时，它就相当于一把互斥锁。

---

# 十一、atomic 分析

## 48. atomic 是线程安全的吗？

`atomic` 只能保证属性 getter 和 setter 方法本身是原子性的，不能保证整个使用过程是线程安全的。

```objc
@property (atomic, assign) NSInteger count;
```

它能保证：

```objc
self.count = 10;
NSInteger value = self.count;
```

单次 setter/getter 是安全的。

但是下面这种不是安全的：

```objc
self.count = self.count + 1;
```

这行代码不是一个原子操作，它至少包含：

1. get count
2. count + 1
3. set count

多个线程同时执行时，仍然可能出现数据错乱。

所以：atomic 不等于线程安全。

---

## 49. nonatomic 和 atomic 有什么区别？

### nonatomic

```objc
@property (nonatomic, copy) NSString *name;
```

特点：

1. 不加锁
2. 性能更好
3. 非线程安全
4. iOS 开发中更常用

### atomic

```objc
@property (atomic, copy) NSString *name;
```

特点：

1. getter/setter 内部加锁
2. 单次读写是原子性的
3. 性能较差
4. 不能保证整个对象逻辑线程安全

iOS 开发中通常使用 `nonatomic`，需要线程安全时自己设计同步方案。

---

# 十二、读写安全

## 50. 什么是多读单写？

多读单写是指：

1. 同一时间允许多个线程读
2. 同一时间只允许一个线程写
3. 读和写不能同时进行
4. 写和写不能同时进行

这种场景常见于：

1. 文件读写
2. 缓存读写
3. 数据库读写
4. 配置表读取
5. 内存数据源访问

因为读操作不会修改数据，所以多个读可以并发。
写操作会修改数据，所以写必须互斥。

---

## 51. iOS 中如何实现多读单写？

常见方案有两种：

1. pthread_rwlock
2. dispatch_barrier_async

---

## 52. pthread_rwlock 如何实现多读单写？

```objc
#import <pthread.h>

@interface DataManager : NSObject

@property (nonatomic, assign) pthread_rwlock_t lock;
@property (nonatomic, strong) NSMutableDictionary *dict;

@end

@implementation DataManager

- (instancetype)init {
    self = [super init];
    if (self) {
        pthread_rwlock_init(&_lock, NULL);
        self.dict = [NSMutableDictionary dictionary];
    }
    return self;
}

- (id)objectForKey:(NSString *)key {
    pthread_rwlock_rdlock(&_lock);

    id value = self.dict[key];

    pthread_rwlock_unlock(&_lock);

    return value;
}

- (void)setObject:(id)obj forKey:(NSString *)key {
    pthread_rwlock_wrlock(&_lock);

    self.dict[key] = obj;

    pthread_rwlock_unlock(&_lock);
}

- (void)dealloc {
    pthread_rwlock_destroy(&_lock);
}

@end
```

读操作使用：

```objc
pthread_rwlock_rdlock(&_lock);
```

写操作使用：

```objc
pthread_rwlock_wrlock(&_lock);
```

这样可以做到多个线程同时读，但写操作独占。

---

## 53. dispatch_barrier_async 如何实现多读单写？

使用自定义并发队列 + 栅栏函数。

```objc
@interface DataManager : NSObject

@property (nonatomic, strong) dispatch_queue_t queue;
@property (nonatomic, strong) NSMutableDictionary *dict;

@end

@implementation DataManager

- (instancetype)init {
    self = [super init];
    if (self) {
        self.queue = dispatch_queue_create("com.demo.rw", DISPATCH_QUEUE_CONCURRENT);
        self.dict = [NSMutableDictionary dictionary];
    }
    return self;
}

- (id)objectForKey:(NSString *)key {
    __block id value = nil;

    dispatch_sync(self.queue, ^{
        value = self.dict[key];
    });

    return value;
}

- (void)setObject:(id)obj forKey:(NSString *)key {
    dispatch_barrier_async(self.queue, ^{
        self.dict[key] = obj;
    });
}

@end
```

读操作使用 `dispatch_sync`，可以并发读。
写操作使用 `dispatch_barrier_async`，写的时候会拦住前后的任务。

执行效果：

```objc
读 读 读 写 读 读 读
```

写任务执行时：

1. 前面的读任务先执行完
2. 写任务独占执行
3. 写任务执行完后，后面的读任务继续并发执行

---

## 54. dispatch_barrier_async 使用时有什么注意点？

最重要的一点：

`dispatch_barrier_async` 必须配合自己创建的并发队列使用。

正确：

```objc
dispatch_queue_t queue = dispatch_queue_create("com.demo.concurrent", DISPATCH_QUEUE_CONCURRENT);

dispatch_barrier_async(queue, ^{
    NSLog(@"写操作");
});
```

错误或无效场景：

```objc
dispatch_barrier_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"写操作");
});
```

如果传入的是全局并发队列，栅栏效果无效，基本等同于普通 `dispatch_async`。

如果传入的是串行队列，也没有多读单写意义。

---

# 十三、死锁综合题

## 55. 什么是死锁？

死锁是指多个任务或线程互相等待对方完成，导致所有任务都无法继续执行。

常见死锁场景：

1. 主线程同步派发任务到主队列
2. 当前串行队列同步派发任务到当前队列
3. 多把锁交叉等待
4. 加锁后异常路径没有解锁
5. 递归调用普通锁

---

## 56. 主线程同步派发到主队列为什么死锁？

```objc
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"执行任务");
});
```

如果这段代码在主线程执行，就会死锁。

原因：

1. 主线程正在执行当前代码
2. `dispatch_sync` 等待 block 执行完
3. block 被添加到主队列
4. 主队列任务必须在主线程执行
5. 主线程被 sync 卡住，无法执行 block
6. block 无法执行，sync 无法返回

---

## 57. 如何避免 GCD 死锁？

可以遵循以下原则：

1. 不要在主线程中 `dispatch_sync` 到主队列
2. 不要在当前串行队列中 `dispatch_sync` 到当前队列
3. 能用 `dispatch_async` 就不要乱用 `dispatch_sync`
4. 需要回主线程刷新 UI 时，用 `dispatch_async`
5. 使用锁时避免交叉加锁
6. 加锁范围尽量小

正确写法：

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"刷新 UI");
});
```

---

# 十四、项目实战题

## 58. 网络请求完成后为什么要回到主线程刷新 UI？

因为 UIKit 不是线程安全的，UI 相关操作必须在主线程执行。

错误写法：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    self.label.text = @"请求完成";
});
```

正确写法：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSString *result = @"请求完成";

    dispatch_async(dispatch_get_main_queue(), ^{
        self.label.text = result;
    });
});
```

---

## 59. 图片下载、解码、显示应该如何做？

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSData *data = [NSData dataWithContentsOfURL:url];
    UIImage *image = [UIImage imageWithData:data];

    dispatch_async(dispatch_get_main_queue(), ^{
        self.imageView.image = image;
    });
});
```

优化方向：

1. 下载放子线程
2. 图片解码放子线程
3. UI 赋值回主线程
4. 做缓存，避免重复下载
5. 控制并发数量，避免同时下载太多图片

---

## 60. 多个网络请求全部完成后刷新页面，怎么做？

可以使用 dispatch_group。

```objc
dispatch_group_t group = dispatch_group_create();

dispatch_group_enter(group);
[self requestUserInfo:^{
    dispatch_group_leave(group);
}];

dispatch_group_enter(group);
[self requestOrderList:^{
    dispatch_group_leave(group);
}];

dispatch_group_enter(group);
[self requestConfig:^{
    dispatch_group_leave(group);
}];

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    [self.tableView reloadData];
});
```

---

# 十五、性能排序题

## 61. iOS 线程同步方案性能大概如何排序？

PPT 中给出的性能从高到低大致为：

```objc
os_unfair_lock
OSSpinLock
dispatch_semaphore
pthread_mutex
dispatch_queue(DISPATCH_QUEUE_SERIAL)
NSLock
NSCondition
pthread_mutex(recursive)
NSRecursiveLock
NSConditionLock
@synchronized
```

实际回答时要注意：

虽然 OSSpinLock 性能高，但已经不安全，不推荐使用。

推荐优先考虑：

1. os_unfair_lock
2. pthread_mutex
3. dispatch_semaphore
4. dispatch_queue
5. NSLock

如果只是普通业务同步，NSLock、dispatch_semaphore、串行队列都可以。
如果追求性能，可以考虑 os_unfair_lock。
如果是读多写少场景，优先考虑 pthread_rwlock 或 dispatch_barrier_async。

---

# 十六、高频总结

## 62. iOS 多线程方案总结

iOS 中常见多线程方案有 pthread、NSThread、GCD、NSOperationQueue。

pthread 最底层，跨平台，但使用复杂。
NSThread 是 OC 封装，可以直接操作线程对象，但需要自己管理。
GCD 是 Apple 提供的多线程方案，通过队列和任务来调度线程，自动管理线程生命周期，使用频率很高。
NSOperationQueue 基于 GCD，面向对象，支持取消、依赖、最大并发数等高级功能，适合复杂任务管理。

---

## 63. GCD 总结

GCD 的核心是任务和队列。

任务就是 block。
队列决定任务的执行方式。

同步和异步影响是否阻塞当前线程，以及是否具备开启新线程的能力。
串行和并发影响任务的执行顺序。

同步不具备开启新线程能力。
异步具备开启新线程能力。
串行队列任务一个接一个执行。
并发队列任务可以同时执行。

但并发队列只有配合异步函数，才真正具备并发效果。

---

## 64. 线程安全总结

线程安全问题的本质是多个线程同时访问同一份共享资源，导致数据错乱。

解决方案是线程同步，常见方式是加锁。

常见锁包括 os_unfair_lock、pthread_mutex、dispatch_semaphore、NSLock、NSRecursiveLock、NSCondition、NSConditionLock、@synchronized 等。

自旋锁等待时会一直占用 CPU，适合等待时间极短的场景。
互斥锁等待时线程会休眠，适合等待时间较长或竞争激烈的场景。

---

## 65. 读写安全总结

读写安全的典型场景是多读单写。

要求：

1. 多个读可以同时进行
2. 写只能一个线程进行
3. 读写不能同时进行
4. 写写不能同时进行

iOS 中常见实现方式：

1. pthread_rwlock
2. dispatch_barrier_async

使用 dispatch_barrier_async 时，必须使用自己创建的并发队列，不能使用全局并发队列。

---

# 最后背诵版

多线程的核心是把耗时任务放到子线程执行，避免阻塞主线程。iOS 常见方案有 pthread、NSThread、GCD、NSOperationQueue。实际开发中最常用 GCD 和 NSOperationQueue。

GCD 的核心是队列和任务。同步和异步决定是否阻塞当前线程、是否具备开启新线程能力；串行和并发决定任务执行顺序。同步不具备开启新线程能力，异步具备开启新线程能力。串行队列任务一个接一个执行，并发队列任务可以同时执行，但并发队列只有配合异步函数才真正并发。

使用 GCD 要特别注意死锁。主线程同步派发任务到主队列会死锁，当前串行队列同步派发任务到当前队列也会死锁。

线程安全问题的本质是多个线程同时访问共享资源导致数据错乱。解决方案是线程同步，常见方式是加锁。常见锁有 os_unfair_lock、pthread_mutex、dispatch_semaphore、NSLock、NSRecursiveLock、NSCondition、NSConditionLock、@synchronized 等。

自旋锁等待时会一直占用 CPU，适合等待时间很短的场景；互斥锁等待时线程会休眠，适合等待时间较长、临界区复杂或竞争激烈的场景。

atomic 只能保证 getter 和 setter 本身的原子性，不能保证整个使用过程线程安全。

读多写少场景可以使用 pthread_rwlock 或 dispatch_barrier_async。使用 dispatch_barrier_async 时，必须传入自己创建的并发队列，不能传全局并发队列。
