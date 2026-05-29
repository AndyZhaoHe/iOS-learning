# iOS Runtime 题库与详细答案

来源：根据用户上传的 `02-Runtime.pptx` 整理。

---

# 一、Runtime 基础概念

## 1. 什么是 Runtime？

**答案：**

Runtime 是 Objective-C 运行时系统，本质是一套由 C/C++/汇编实现的底层 API。Objective-C 之所以具备动态特性，就是依赖 Runtime 来完成的。

Objective-C 和 C、C++ 最大的区别之一是：

C/C++ 很多行为在编译期就确定了，而 Objective-C 允许很多事情推迟到运行时决定。

比如：

```objc
[person run];
```

表面上看是对象调用方法，底层实际上会被转换成：

```objc
objc_msgSend(person, @selector(run));
```

也就是说，Objective-C 的方法调用，本质是“给对象发送消息”。

Runtime 支撑了这些能力：

- 方法查找
- 消息发送
- 动态添加方法
- 动态创建类
- 方法交换
- 关联对象
- 消息转发
- KVC / KVO 底层机制
- 字典转模型
- 自动归档解档
- 分类添加属性

**追问点：Runtime 平时项目中用过吗？**

可以这样答：

项目中常见使用场景有：

1. **分类添加属性**

分类本身不能直接添加成员变量，但可以通过关联对象实现“类似属性”的效果。

2. **字典转模型**

通过 Runtime 遍历类的属性列表或成员变量列表，再结合 KVC 给模型赋值。

3. **自动归档解档**

遍历成员变量，自动实现 `encodeWithCoder:` 和 `initWithCoder:`。

4. **方法交换**

例如拦截系统方法、统计按钮点击、替换 `viewDidAppear:` 做埋点。

5. **防止找不到方法崩溃**

利用消息转发机制处理 `unrecognized selector sent to instance`。

---

# 二、OC 消息机制

## 2. OC 的方法调用底层是什么？

**答案：**

Objective-C 的方法调用本质是消息发送。

例如：

```objc
[person test];
```

底层会变成：

```objc
objc_msgSend(person, @selector(test));
```

其中：

- `person` 是消息接收者，也叫 receiver
- `test` 是方法名，对应 SEL
- `objc_msgSend` 会根据 receiver 的 isa 找到对应的 Class
- 然后从方法缓存、方法列表、父类中查找方法实现 IMP
- 找到后调用 IMP
- 找不到则进入动态方法解析
- 动态解析失败后进入消息转发

---

## 3. objc_msgSend 的执行流程是什么？

**答案：**

`objc_msgSend` 主要分为三大阶段：

```text
消息发送
↓
动态方法解析
↓
消息转发
```

详细流程：

### 第一阶段：消息发送

调用：

```objc
[person test];
```

底层：

```objc
objc_msgSend(person, @selector(test));
```

执行流程：

1. 判断 receiver 是否为 nil
2. 如果 receiver 为 nil，直接返回，不崩溃
3. 如果 receiver 不为 nil，通过 isa 找到 receiver 的 Class
4. 先从当前类的 cache 中查找方法
5. 如果 cache 命中，直接调用 IMP
6. 如果 cache 没找到，从当前类的 method list 中查找
7. 如果当前类找到，调用 IMP，并缓存到 cache
8. 如果当前类没找到，继续去父类查找
9. 父类同样先查 cache，再查 method list
10. 一直查到 NSObject
11. 如果都找不到，进入动态方法解析

### 第二阶段：动态方法解析

Runtime 会给类一次“补救”的机会。

实例方法对应：

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```

类方法对应：

```objc
+ (BOOL)resolveClassMethod:(SEL)sel;
```

可以在这里动态添加方法实现：

```objc
void c_other(id self, SEL _cmd) {
    NSLog(@"动态添加了 other 方法");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(other)) {
        class_addMethod(self,
                        sel,
                        (IMP)c_other,
                        "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

如果动态添加成功，Runtime 会重新走一遍消息发送流程。

### 第三阶段：消息转发

如果动态方法解析也失败，就进入消息转发。

消息转发分两步：

#### 快速转发

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

如果返回一个非 nil 对象，Runtime 会把消息转发给这个对象。

例如：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(test)) {
        return [[OtherObject alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

等价于：

```objc
objc_msgSend(otherObject, @selector(test));
```

#### 慢速转发

如果快速转发返回 nil，就会走：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
```

如果返回 nil，程序崩溃：

```text
unrecognized selector sent to instance
```

如果返回非 nil，会继续调用：

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

示例：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(test)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"可以在这里处理找不到的方法");
}
```

---

## 4. 给 nil 对象发送消息会发生什么？

**答案：**

不会崩溃。

例如：

```objc
MJPerson *person = nil;
[person test];
```

底层是：

```objc
objc_msgSend(nil, @selector(test));
```

`objc_msgSend` 一开始会判断 receiver 是否为 nil，如果是 nil，直接返回。

所以 Objective-C 中给 nil 发消息是安全的。

但是要注意返回值：

```objc
NSString *name = [person name];
```

如果 `person` 为 nil，返回值是 nil。

```objc
NSInteger age = [person age];
```

如果返回的是基本数据类型，通常返回 0。

```objc
CGRect frame = [view frame];
```

如果是结构体，情况和架构、ABI 有关，不建议依赖这种行为。

---

# 三、isa 指针

## 5. isa 是什么？

**答案：**

每个 Objective-C 对象内部都有一个 isa。

isa 用来指向对象所属的类。

例如：

```objc
MJPerson *person = [[MJPerson alloc] init];
```

关系是：

```text
person 对象的 isa → MJPerson 类对象
```

类对象也有 isa：

```text
MJPerson 类对象的 isa → MJPerson 元类对象
```

元类对象也有 isa，最终会形成一套完整的 isa 继承体系。

---

## 6. 对象、类对象、元类对象分别存什么？

**答案：**

### 实例对象

实例对象主要存：

- isa
- 成员变量的值

例如：

```objc
@interface MJPerson : NSObject {
    int _age;
}
@property(nonatomic, copy) NSString *name;
@end
```

实例对象里可能存：

```text
isa
_age
_name
```

实例对象不存方法。

### 类对象

类对象主要存：

- isa
- superclass
- 类的实例方法
- 属性信息
- 协议信息
- 成员变量信息

也就是说：

```objc
- (void)test;
```

这种实例方法存在类对象中。

### 元类对象

元类对象主要存：

- isa
- superclass
- 类方法

例如：

```objc
+ (void)test;
```

这种类方法存在元类对象中。

---

## 7. isa 在 arm64 之前和之后有什么区别？

**答案：**

arm64 之前，isa 基本就是一个普通指针，直接存储 Class 或 Meta-Class 的地址。

arm64 之后，isa 被优化成了一个 union 结构，不再只是单纯的指针，而是通过位域存储更多信息。

常见字段有：

```text
nonpointer
has_assoc
has_cxx_dtor
shiftcls
magic
weakly_referenced
deallocating
extra_rc
has_sidetable_rc
```

---

## 8. isa 位域中各字段含义是什么？

**答案：**

### nonpointer

表示 isa 是否是优化过的 isa。

```text
0：普通指针
1：优化后的 isa，使用位域存储更多信息
```

### has_assoc

表示对象是否有关联对象。

如果没有关联对象，对象释放时可以更快。

### has_cxx_dtor

表示对象是否有 C++ 析构函数，或者 Objective-C 的 `.cxx_destruct`。

如果没有，释放时更快。

### shiftcls

存储 Class 或 Meta-Class 的地址信息。

因为对象地址通常是内存对齐的，低几位固定为 0，所以 Runtime 可以利用这些位存储额外信息。

### magic

用于调试，帮助判断对象是否已经完成初始化。

### weakly_referenced

表示对象是否被弱引用过。

如果对象从未被 weak 指针引用过，释放时会更快。

### deallocating

表示对象是否正在释放。

### extra_rc

存储引用计数的一部分。

注意：这里存的是引用计数减 1。

### has_sidetable_rc

如果对象引用计数过大，isa 中存不下，就会存到 SideTable 中。

此字段表示引用计数是否额外存储在 SideTable 中。

---

# 四、Class 结构

## 9. Class 内部结构大概是什么？

**答案：**

Class 内部大概包含：

```text
isa
superclass
cache
bits
```

其中：

```text
bits & FAST_DATA_MASK → class_rw_t
```

`class_rw_t` 中有：

- methods
- properties
- protocols

这些是可读可写的。

`class_rw_t` 还会关联到 `class_ro_t`。

`class_ro_t` 中有：

- baseMethodList
- baseProtocols
- ivars
- baseProperties

这些是只读的，表示类最初编译时确定下来的内容。

---

## 10. class_rw_t 和 class_ro_t 有什么区别？

**答案：**

### class_ro_t

`ro` 可以理解为 read only，只读。

它保存类在编译期确定的原始信息：

```text
baseMethodList
baseProtocols
ivars
baseProperties
```

比如类本身声明的方法、属性、成员变量等。

### class_rw_t

`rw` 可以理解为 read write，可读可写。

它保存运行时可以变化的内容：

```text
methods
properties
protocols
```

比如分类中的方法、动态添加的方法，都会进入 `class_rw_t`。

---

## 11. 分类的方法存在哪里？

**答案：**

分类的方法会合并到类对象的 `class_rw_t` 的方法列表中。

所以分类可以添加方法。

但是分类不能直接添加成员变量，因为成员变量布局在类注册后就固定了。

如果分类想添加“属性”，只能通过关联对象实现。

---

# 五、SEL、IMP、Method

## 12. SEL 是什么？

**答案：**

SEL 是方法名，也叫选择器。

底层可以理解为类似 `char *` 的结构。

可以通过下面方式获得：

```objc
SEL sel1 = @selector(test);
SEL sel2 = sel_registerName("test");
```

可以转成字符串：

```objc
NSStringFromSelector(sel1);
sel_getName(sel1);
```

重要点：

不同类中，只要方法名相同，对应的 SEL 就相同。

例如：

```objc
[Person test];
[Dog test];
```

这两个 `test` 的 SEL 是同一个。

---

## 13. IMP 是什么？

**答案：**

IMP 是方法的具体实现，本质是函数指针。

定义大概类似：

```objc
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);
```

也就是说，Objective-C 方法最终都会变成类似 C 函数的调用。

例如：

```objc
- (void)test {
    NSLog(@"test");
}
```

底层可以理解为：

```objc
void test(id self, SEL _cmd) {
    NSLog(@"test");
}
```

每个 OC 方法默认都有两个隐藏参数：

```objc
self
_cmd
```

---

## 14. Method 是什么？

**答案：**

Method 是 Runtime 对方法的封装。

一个 Method 主要包含：

```text
SEL name
const char *types
IMP imp
```

也就是：

```text
方法名
方法类型编码
方法实现
```

例如：

```objc
Method method = class_getInstanceMethod([MJPerson class], @selector(test));
```

可以获取：

```objc
SEL name = method_getName(method);
IMP imp = method_getImplementation(method);
const char *types = method_getTypeEncoding(method);
```

---

## 15. Type Encoding 是什么？

**答案：**

Type Encoding 是 Runtime 用字符串描述方法返回值和参数类型的机制。

比如：

```objc
- (void)test;
```

类型编码通常是：

```text
v@:
```

含义：

```text
v   返回值 void
@   self，表示对象
:   _cmd，表示 SEL
```

例如：

```objc
- (int)test:(NSString *)name age:(int)age;
```

类型编码可能类似：

```text
i@: @ i
```

常见编码：

```text
v    void
@    object
:    SEL
i    int
q    long long / NSInteger
f    float
d    double
B    BOOL
```

可以用：

```objc
@encode(int)
@encode(NSString *)
@encode(CGRect)
```

获取类型编码。

---

# 六、方法缓存

## 16. Runtime 为什么要有方法缓存？

**答案：**

因为 Objective-C 方法调用是动态查找的，如果每次调用都去方法列表里查，会影响性能。

所以 Class 内部有一个 `cache_t`，用于缓存曾经调用过的方法。

方法第一次调用时：

```text
cache 找不到
↓
方法列表查找
↓
找到 IMP
↓
调用 IMP
↓
把 SEL 和 IMP 缓存到 cache
```

下次再调用同一个方法：

```text
直接从 cache 找到 IMP
↓
调用
```

这样可以提高方法调用效率。

---

## 17. 方法缓存底层是什么结构？

**答案：**

方法缓存底层类似哈希表。

里面有很多 bucket。

每个 bucket 存：

```text
SEL
IMP
```

大致结构：

```text
bucket_t {
    SEL _key;
    IMP _imp;
}
```

查找时类似：

```text
index = SEL & mask
```

如果发生冲突，会继续探测下一个位置。

这种设计是典型的“空间换时间”。

---

# 七、动态方法解析

## 18. 什么是动态方法解析？

**答案：**

当对象收到一个找不到的方法时，Runtime 不会马上崩溃，而是先给开发者一次机会，让开发者可以动态添加方法实现。

实例方法：

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```

类方法：

```objc
+ (BOOL)resolveClassMethod:(SEL)sel;
```

如果在这里用 `class_addMethod` 添加了方法，并返回 YES，那么 Runtime 会重新执行消息发送流程。

---

## 19. 动态添加实例方法怎么写？

**答案：**

```objc
#import <objc/runtime.h>

void runImp(id self, SEL _cmd) {
    NSLog(@"%@ - %@", self, NSStringFromSelector(_cmd));
}

@implementation MJPerson

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(run)) {
        class_addMethod(self,
                        sel,
                        (IMP)runImp,
                        "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

@end
```

调用：

```objc
MJPerson *person = [[MJPerson alloc] init];
[person performSelector:@selector(run)];
```

执行过程：

```text
找不到 run
↓
调用 resolveInstanceMethod:
↓
动态添加 run 的实现
↓
重新走消息发送
↓
调用 runImp
```

---

## 20. 动态添加类方法应该加到哪里？

**答案：**

类方法存储在元类对象中。

所以动态添加类方法时，要把方法添加到元类对象中。

示例：

```objc
void classRunImp(id self, SEL _cmd) {
    NSLog(@"类方法 run");
}

+ (BOOL)resolveClassMethod:(SEL)sel {
    if (sel == @selector(run)) {
        Class metaClass = object_getClass(self);
        class_addMethod(metaClass,
                        sel,
                        (IMP)classRunImp,
                        "v@:");
        return YES;
    }
    return [super resolveClassMethod:sel];
}
```

重点：

```objc
Class metaClass = object_getClass(self);
```

因为 `self` 在类方法中是类对象，`object_getClass(self)` 才能拿到元类对象。

---

# 八、消息转发

## 21. 消息转发完整流程是什么？

**答案：**

如果消息发送和动态解析都失败，就进入消息转发。

流程是：

```text
forwardingTargetForSelector:
↓
methodSignatureForSelector:
↓
forwardInvocation:
↓
doesNotRecognizeSelector:
```

详细说明：

1. 先调用：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

如果返回非 nil 对象，消息转发给这个对象。

2. 如果返回 nil，调用：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
```

如果返回 nil，调用：

```objc
doesNotRecognizeSelector:
```

然后崩溃。

3. 如果返回方法签名，则创建 NSInvocation，并调用：

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

可以在这里自定义处理逻辑。

---

## 22. forwardingTargetForSelector 和 forwardInvocation 有什么区别？

**答案：**

### forwardingTargetForSelector

属于快速转发。

特点：

- 只能把消息转给另一个对象
- 不能修改参数
- 不能自定义复杂逻辑
- 性能相对更好

示例：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(run)) {
        return self.dog;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

### forwardInvocation

属于慢速转发。

特点：

- 可以修改 target
- 可以修改 selector
- 可以修改参数
- 可以拿到返回值
- 可以做复杂逻辑
- 性能比快速转发低

示例：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(run)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;

    if ([self.dog respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:self.dog];
    }
}
```

---

## 23. 如何利用消息转发防止找不到方法崩溃？

**答案：**

可以在 `methodSignatureForSelector:` 中返回一个兜底方法签名，然后在 `forwardInvocation:` 中统一处理。

示例：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"方法 %@ 没有实现，已被兜底处理",
          NSStringFromSelector(anInvocation.selector));
}
```

不过这种做法不建议滥用。

更好的方式是：

- 只处理特定 selector
- 打日志
- 上报异常
- 避免掩盖真正 bug

---

# 九、super 的本质

## 24. super 是什么？super 调用底层是什么？

**答案：**

`super` 不是对象。

`super` 只是编译器关键字。

当写：

```objc
[super viewDidLoad];
```

底层会转换成：

```objc
objc_msgSendSuper2(...)
```

它会构造一个结构体：

```objc
struct objc_super {
    id receiver;
    Class super_class;
};
```

在较新 Runtime 中是类似 `objc_super2` 的结构。

其中：

```text
receiver：仍然是当前对象 self
current_class：当前类
```

注意：

`super` 调用时，消息接收者仍然是 `self`，只是查找方法时从父类开始查找。

---

## 25. [self class] 和 [super class] 打印结果一样吗？

**答案：**

通常一样。

示例：

```objc
@implementation MJPerson

- (void)test {
    NSLog(@"%@", [self class]);
    NSLog(@"%@", [super class]);
}

@end
```

如果对象是 `MJPerson` 实例，那么两行通常都打印：

```text
MJPerson
MJPerson
```

原因：

```objc
[super class]
```

底层不是把消息发给父类对象，而是：

```text
receiver 仍然是 self
查找方法从父类开始
```

而 `class` 方法内部返回的是 receiver 的真实类。

所以结果仍然是 `MJPerson`。

---

# 十、方法交换

## 26. 什么是方法交换？

**答案：**

方法交换就是交换两个方法的 IMP。

常用 API：

```objc
method_exchangeImplementations(Method m1, Method m2);
```

例如：

```objc
Method m1 = class_getInstanceMethod([UIButton class],
                                    @selector(sendAction:to:forEvent:));

Method m2 = class_getInstanceMethod([UIButton class],
                                    @selector(mj_sendAction:to:forEvent:));

method_exchangeImplementations(m1, m2);
```

交换前：

```text
sendAction:to:forEvent:     → 原实现
mj_sendAction:to:forEvent:  → 自定义实现
```

交换后：

```text
sendAction:to:forEvent:     → 自定义实现
mj_sendAction:to:forEvent:  → 原实现
```

---

## 27. 方法交换的常见用途有哪些？

**答案：**

常见用途：

1. 统计按钮点击
2. 页面曝光埋点
3. 防止数组越界崩溃
4. 防止字典插入 nil 崩溃
5. 替换系统方法
6. AOP 切面编程
7. Debug 阶段排查问题

例如按钮点击统计：

```objc
@implementation UIButton (MJ)

+ (void)load {
    Method m1 = class_getInstanceMethod(self,
        @selector(sendAction:to:forEvent:));

    Method m2 = class_getInstanceMethod(self,
        @selector(mj_sendAction:to:forEvent:));

    method_exchangeImplementations(m1, m2);
}

- (void)mj_sendAction:(SEL)action
                   to:(id)target
             forEvent:(UIEvent *)event {
    NSLog(@"按钮点击统计");
    [self mj_sendAction:action to:target forEvent:event];
}

@end
```

注意：

交换后：

```objc
[self mj_sendAction:action to:target forEvent:event];
```

实际调用的是系统原来的 `sendAction:to:forEvent:`。

---

## 28. 方法交换为什么一般写在 +load 中？

**答案：**

因为 `+load` 在类或分类被加载到内存时就会调用，时机非常早。

这样可以保证方法在使用之前已经完成交换。

不过实际开发中要注意：

- `+load` 调用时机早，可能影响启动性能
- 多个分类都交换同一个方法时，顺序不可控
- 应使用 `dispatch_once` 保证只交换一次

示例：

```objc
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Method m1 = class_getInstanceMethod(self, @selector(viewDidAppear:));
        Method m2 = class_getInstanceMethod(self, @selector(mj_viewDidAppear:));
        method_exchangeImplementations(m1, m2);
    });
}
```

---

# 十一、关联对象

## 29. 分类为什么不能直接添加成员变量？

**答案：**

因为类的成员变量布局在编译期基本已经确定，并且类注册后，实例对象的内存布局就固定了。

分类是在运行时把方法、协议、属性等信息合并到类中，但不能改变实例对象的内存布局。

所以分类可以添加：

```text
方法
协议
属性声明
```

但不能真正添加：

```text
成员变量
```

如果分类中写：

```objc
@property(nonatomic, copy) NSString *name;
```

编译器只会生成 getter / setter 的声明，不会自动生成成员变量，也不会自动生成方法实现。

---

## 30. 如何给分类添加属性？

**答案：**

通过关联对象。

常用 API：

```objc
objc_setAssociatedObject(id object,
                         const void *key,
                         id value,
                         objc_AssociationPolicy policy);

objc_getAssociatedObject(id object,
                         const void *key);

objc_removeAssociatedObjects(id object);
```

示例：

```objc
#import <objc/runtime.h>

@implementation NSObject (MJ)

static const void *MJNameKey = &MJNameKey;

- (void)setMj_name:(NSString *)mj_name {
    objc_setAssociatedObject(self,
                             MJNameKey,
                             mj_name,
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)mj_name {
    return objc_getAssociatedObject(self, MJNameKey);
}

@end
```

---

## 31. 关联对象有哪些策略？

**答案：**

常见策略：

```text
OBJC_ASSOCIATION_ASSIGN
OBJC_ASSOCIATION_RETAIN_NONATOMIC
OBJC_ASSOCIATION_COPY_NONATOMIC
OBJC_ASSOCIATION_RETAIN
OBJC_ASSOCIATION_COPY
```

对应关系类似属性修饰符：

```text
assign
strong nonatomic
copy nonatomic
strong atomic
copy atomic
```

常用：

```objc
OBJC_ASSOCIATION_COPY_NONATOMIC
OBJC_ASSOCIATION_RETAIN_NONATOMIC
```

---

# 十二、KVC、字典转模型、自动归档

## 32. Runtime 如何实现字典转模型？

**答案：**

核心思路：

1. 利用 Runtime 获取模型类的属性列表
2. 遍历属性名
3. 从字典中取对应 value
4. 使用 KVC 给模型赋值

示例：

```objc
+ (instancetype)mj_modelWithDict:(NSDictionary *)dict {
    id obj = [[self alloc] init];

    unsigned int count = 0;
    objc_property_t *properties = class_copyPropertyList(self, &count);

    for (int i = 0; i < count; i++) {
        objc_property_t property = properties[i];

        const char *name = property_getName(property);
        NSString *key = [NSString stringWithUTF8String:name];

        id value = dict[key];

        if (value && value != [NSNull null]) {
            [obj setValue:value forKey:key];
        }
    }

    free(properties);
    return obj;
}
```

注意点：

- `class_copyPropertyList` 返回的数组需要 `free`
- 要处理 `NSNull`
- 要处理 key 不一致
- 要处理嵌套模型
- 要处理数组里面的模型
- 要处理类型转换

---

## 33. Runtime 如何实现自动归档解档？

**答案：**

核心是遍历成员变量。

### 归档

```objc
- (void)encodeWithCoder:(NSCoder *)coder {
    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([self class], &count);

    for (int i = 0; i < count; i++) {
        Ivar ivar = ivars[i];

        const char *name = ivar_getName(ivar);
        NSString *key = [NSString stringWithUTF8String:name];

        id value = [self valueForKey:key];

        [coder encodeObject:value forKey:key];
    }

    free(ivars);
}
```

### 解档

```objc
- (instancetype)initWithCoder:(NSCoder *)coder {
    if (self = [super init]) {
        unsigned int count = 0;
        Ivar *ivars = class_copyIvarList([self class], &count);

        for (int i = 0; i < count; i++) {
            Ivar ivar = ivars[i];

            const char *name = ivar_getName(ivar);
            NSString *key = [NSString stringWithUTF8String:name];

            id value = [coder decodeObjectForKey:key];

            [self setValue:value forKey:key];
        }

        free(ivars);
    }
    return self;
}
```

---

# 十三、Runtime API 高频题

## 34. 如何获取一个类的所有成员变量？

**答案：**

```objc
unsigned int count = 0;
Ivar *ivars = class_copyIvarList([MJPerson class], &count);

for (int i = 0; i < count; i++) {
    Ivar ivar = ivars[i];

    const char *name = ivar_getName(ivar);
    const char *type = ivar_getTypeEncoding(ivar);

    NSLog(@"%s - %s", name, type);
}

free(ivars);
```

重点：

带 `copy` 的 Runtime API，返回的 C 数组通常需要手动 `free`。

---

## 35. 如何获取一个类的所有属性？

**答案：**

```objc
unsigned int count = 0;
objc_property_t *properties = class_copyPropertyList([MJPerson class], &count);

for (int i = 0; i < count; i++) {
    objc_property_t property = properties[i];

    const char *name = property_getName(property);
    const char *attrs = property_getAttributes(property);

    NSLog(@"%s - %s", name, attrs);
}

free(properties);
```

---

## 36. 如何获取一个类的所有方法？

**答案：**

```objc
unsigned int count = 0;
Method *methods = class_copyMethodList([MJPerson class], &count);

for (int i = 0; i < count; i++) {
    Method method = methods[i];

    SEL sel = method_getName(method);
    IMP imp = method_getImplementation(method);
    const char *types = method_getTypeEncoding(method);

    NSLog(@"%@ - %p - %s",
          NSStringFromSelector(sel),
          imp,
          types);
}

free(methods);
```

---

## 37. object_getClass 和 [obj class] 有什么区别？

**答案：**

### `[obj class]`

是调用对象的 `class` 方法。

通常返回对象所属的类。

```objc
MJPerson *person = [[MJPerson alloc] init];
NSLog(@"%@", [person class]);
```

结果：

```text
MJPerson
```

### `object_getClass(obj)`

是真正从 Runtime 层面获取 obj 的 isa 指向。

```objc
object_getClass(person)
```

返回：

```text
MJPerson 类对象
```

如果传入类对象：

```objc
object_getClass([MJPerson class])
```

返回：

```text
MJPerson 元类对象
```

所以：

```objc
[NSObject class]
```

返回 NSObject 类对象。

```objc
object_getClass([NSObject class])
```

返回 NSObject 元类对象。

---

## 38. class 和 object_getClass 常见打印题

代码：

```objc
NSLog(@"%d", object_isClass([MJPerson class]));
NSLog(@"%d", class_isMetaClass(object_getClass([MJPerson class])));
```

答案：

```text
1
1
```

原因：

```objc
[MJPerson class]
```

是类对象，所以 `object_isClass` 为 YES。

```objc
object_getClass([MJPerson class])
```

拿到的是元类对象，所以 `class_isMetaClass` 为 YES。

---

# 十四、经典代码题

## 39. 以下代码能否执行成功？

```objc
NSObject *obj = [[NSObject alloc] init];
[obj setValue:@"123" forKey:@"name"];
NSLog(@"%@", [obj valueForKey:@"name"]);
```

**答案：**

不能正常执行。

原因：

NSObject 没有 `name` 属性，也没有 `_name`、`_isName`、`name`、`isName` 等相关成员变量。

KVC 找不到对应 key，会调用：

```objc
- (void)setValue:(id)value forUndefinedKey:(NSString *)key;
```

默认实现会抛异常。

异常类似：

```text
[<NSObject ...> setValue:forUndefinedKey:]:
this class is not key value coding-compliant for the key name.
```

---

## 40. 以下代码结果是什么？

```objc
MJPerson *person = [[MJPerson alloc] init];
NSLog(@"%@", [person class]);
NSLog(@"%@", [MJPerson class]);
NSLog(@"%@", object_getClass(person));
NSLog(@"%@", object_getClass([MJPerson class]));
```

假设类名是 `MJPerson`。

答案：

```text
MJPerson
MJPerson
MJPerson
MJPerson
```

但最后一个虽然打印名字也是 `MJPerson`，实际是 `MJPerson` 的元类对象。

解释：

```objc
[person class]
```

返回 person 所属的类对象。

```objc
[MJPerson class]
```

返回 MJPerson 类对象。

```objc
object_getClass(person)
```

返回 person 的 isa，也就是 MJPerson 类对象。

```objc
object_getClass([MJPerson class])
```

返回 MJPerson 类对象的 isa，也就是 MJPerson 元类对象。

---

## 41. isKindOfClass 和 isMemberOfClass 区别？

**答案：**

### isMemberOfClass

只判断对象是否“正好”是某个类的实例。

不包含子类。

```objc
[person isMemberOfClass:[MJPerson class]]
```

只有 person 的真实类就是 MJPerson，才返回 YES。

### isKindOfClass

判断对象是否是某个类或其子类的实例。

包含继承关系。

```objc
[student isKindOfClass:[MJPerson class]]
```

如果 Student 继承自 MJPerson，返回 YES。

示例：

```objc
@interface MJPerson : NSObject
@end

@interface MJStudent : MJPerson
@end

MJStudent *stu = [[MJStudent alloc] init];

NSLog(@"%d", [stu isMemberOfClass:[MJStudent class]]);
NSLog(@"%d", [stu isMemberOfClass:[MJPerson class]]);
NSLog(@"%d", [stu isKindOfClass:[MJStudent class]]);
NSLog(@"%d", [stu isKindOfClass:[MJPerson class]]);
NSLog(@"%d", [stu isKindOfClass:[NSObject class]]);
```

结果：

```text
1
0
1
1
1
```

---

## 42. 类对象调用 isKindOfClass / isMemberOfClass 怎么判断？

这类题容易混淆，因为类对象本身也是对象。

例如：

```objc
NSLog(@"%d", [[NSObject class] isKindOfClass:[NSObject class]]);
NSLog(@"%d", [[NSObject class] isMemberOfClass:[NSObject class]]);
NSLog(@"%d", [[MJPerson class] isKindOfClass:[MJPerson class]]);
NSLog(@"%d", [[MJPerson class] isMemberOfClass:[MJPerson class]]);
```

典型结果：

```text
1
0
0
0
```

解释：

### `[[NSObject class] isKindOfClass:[NSObject class]]`

`[NSObject class]` 是 NSObject 类对象。

类对象的 isa 指向 NSObject 元类。

元类的 superclass 最终会走到 NSObject 类对象。

所以结果为 YES。

### `[[NSObject class] isMemberOfClass:[NSObject class]]`

`isMemberOfClass` 要求 receiver 的 isa 正好等于传入类。

NSObject 类对象的 isa 是 NSObject 元类，不是 NSObject 类对象。

所以为 NO。

### `[[MJPerson class] isKindOfClass:[MJPerson class]]`

MJPerson 类对象的 isa 是 MJPerson 元类。

沿着元类 superclass 链往上找，不会找到 MJPerson 类对象本身。

所以通常是 NO。

### `[[MJPerson class] isMemberOfClass:[MJPerson class]]`

MJPerson 类对象的 isa 是 MJPerson 元类，不是 MJPerson 类对象。

所以是 NO。

---

# 十五、综合高频题

## 43. Runtime 消息发送和消息转发的区别？

**答案：**

消息发送是正常找方法的过程。

```text
cache
↓
当前类方法列表
↓
父类 cache
↓
父类方法列表
```

消息转发是找不到方法后的兜底流程。

```text
forwardingTargetForSelector:
↓
methodSignatureForSelector:
↓
forwardInvocation:
```

区别：

```text
消息发送：找方法并执行
消息转发：找不到方法后，把消息交给其他对象或自定义处理
```

---

## 44. 为什么方法调用要先查 cache？

**答案：**

因为 Objective-C 方法调用非常频繁，如果每次都遍历方法列表，效率会比较低。

方法缓存可以避免重复查找。

第一次调用：

```text
慢：cache miss → method list → IMP
```

后续调用：

```text
快：cache hit → IMP
```

这就是 Runtime 用 cache 提升性能的原因。

---

## 45. class_addMethod、class_replaceMethod、method_exchangeImplementations 区别？

**答案：**

### class_addMethod

添加一个新方法。

如果类中已经有同名方法，添加失败。

```objc
BOOL success = class_addMethod(cls, sel, imp, types);
```

### class_replaceMethod

替换方法实现。

如果方法存在，就替换。

如果方法不存在，就添加。

```objc
class_replaceMethod(cls, sel, imp, types);
```

### method_exchangeImplementations

交换两个已有方法的 IMP。

```objc
method_exchangeImplementations(m1, m2);
```

常见安全写法：

```objc
Method originalMethod = class_getInstanceMethod(cls, originalSelector);
Method swizzledMethod = class_getInstanceMethod(cls, swizzledSelector);

BOOL didAddMethod =
class_addMethod(cls,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

if (didAddMethod) {
    class_replaceMethod(cls,
                        swizzledSelector,
                        method_getImplementation(originalMethod),
                        method_getTypeEncoding(originalMethod));
} else {
    method_exchangeImplementations(originalMethod, swizzledMethod);
}
```

这样可以避免原类没有实现某个方法时直接交换失败。

---

## 46. 为什么类注册后不能再添加成员变量？

**答案：**

因为成员变量影响对象的内存布局。

类注册后，对象实例的大小、成员变量偏移量都已经确定。

如果此时再添加成员变量，会破坏已有对象的内存布局。

所以：

```objc
class_addIvar
```

只能在：

```objc
objc_allocateClassPair
```

之后，

```objc
objc_registerClassPair
```

之前调用。

示例：

```objc
Class cls = objc_allocateClassPair([NSObject class], "MJDog", 0);

class_addIvar(cls, "_age", sizeof(int), log2(sizeof(int)), @encode(int));

objc_registerClassPair(cls);
```

注册之后再加成员变量就会失败。

---

# 十六、最后可以背的总结版

## Runtime 一句话总结

Runtime 是 Objective-C 动态性的底层支撑，它把方法调用转换成消息发送，并在运行时完成方法查找、动态解析、消息转发、类结构管理和方法实现操作。

## objc_msgSend 一句话总结

`objc_msgSend` 会先判断 receiver 是否为 nil，然后从当前类和父类的 cache、方法列表中查找 IMP，找不到就进入动态方法解析，再失败就进入消息转发。

## isa 一句话总结

isa 用来连接对象和类：实例对象的 isa 指向类对象，类对象的 isa 指向元类对象，元类对象中存储类方法。

## SEL / IMP / Method 一句话总结

SEL 是方法名，IMP 是方法实现，Method 是 Runtime 对方法的封装，里面包含 SEL、IMP 和类型编码。

## 方法缓存一句话总结

Runtime 使用 cache_t 缓存已经调用过的方法，底层类似哈希表，用空间换时间，提高消息发送效率。

## 消息转发一句话总结

当方法找不到时，Runtime 会依次尝试快速转发、方法签名、NSInvocation 慢速转发，最终仍无法处理才会调用 `doesNotRecognizeSelector:` 崩溃。

## super 一句话总结

`super` 不是对象，它只是告诉 Runtime 从父类开始查找方法，消息接收者仍然是 self。

---

# 十七、建议重点背这 10 题

1. Runtime 是什么？项目中怎么用？
2. OC 方法调用底层是什么？
3. objc_msgSend 完整流程？
4. isa 指针是什么？arm64 后有什么变化？
5. 对象、类对象、元类对象分别存什么？
6. SEL、IMP、Method 区别？
7. 方法缓存 cache_t 的作用？
8. 动态方法解析怎么实现？
9. 消息转发完整流程？
10. super 的本质是什么？
