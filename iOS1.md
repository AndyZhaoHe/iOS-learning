# iOS Objective-C 语法与 Runtime 题库详解

> 来源：根据用户上传的 `01-OC语法.pptx` 整理。内容覆盖 OC 对象本质、isa、class/meta-class、KVO、KVC、Category、关联对象、Block、__block、循环引用等。

---

## 目录

1. OC 对象本质
2. isa 与 superclass
3. class / meta-class
4. isKindOfClass 与 isMemberOfClass
5. KVO
6. KVC
7. Category
8. +load 与 +initialize
9. 关联对象
10. Block
11. __block
12. 循环引用
13. 代码打印题
14. 快速背诵版

---

# 一、OC 对象本质

## 1. 一个 NSObject 对象占用多少内存？

### 答案

在 64 位环境下，一个 `NSObject` 对象内部真正使用的空间是 **8 字节**，因为它内部主要只有一个 `isa` 指针。

但是系统实际分配的内存通常是 **16 字节**。

```objc
NSObject *obj = [[NSObject alloc] init];

NSLog(@"%zu", class_getInstanceSize([NSObject class]));
NSLog(@"%zu", malloc_size((__bridge const void *)obj));
```

通常输出：

```objc
8
16
```

### 解释

`class_getInstanceSize` 表示创建实例对象至少需要多少内存。

`malloc_size` 表示系统实际分配了多少内存。

所以标准说法：

```text
NSObject 对象内部使用 8 字节，但系统实际分配 16 字节。
```

---

## 2. class_getInstanceSize 和 malloc_size 有什么区别？

### 答案

```objc
class_getInstanceSize([NSObject class])
```

表示：实例对象至少需要多少内存。

```objc
malloc_size((__bridge const void *)obj)
```

表示：系统实际给这个对象分配了多少内存。

### 解释

系统底层分配内存时会考虑内存对齐和分配效率，所以实际分配空间可能大于对象本身需要的空间。

---

## 3. 一个 OC 对象在内存中是如何布局的？

### 答案

OC 对象本质上是 C/C++ 结构体。

一个实例对象的内存通常由两部分组成：

```text
isa 指针
成员变量的具体值
```

例如：

```objc
@interface Student : NSObject {
    int _no;
    int _age;
}
@end
```

64 位下大致布局：

```text
isa   8 字节
_no   4 字节
_age  4 字节
```

总共 16 字节。

---

## 4. Objective-C 对象主要分为哪几种？

### 答案

OC 对象主要分为 3 种：

```text
instance 对象
class 对象
meta-class 对象
```

### instance 对象

通过 `alloc` 创建出来的对象。

```objc
MJPerson *p1 = [[MJPerson alloc] init];
MJPerson *p2 = [[MJPerson alloc] init];
```

`p1` 和 `p2` 是两个不同的实例对象，占用不同内存。

instance 对象中主要存储：

```text
isa 指针
成员变量具体值
```

### class 对象

每个类在内存中有且只有一个 class 对象。

class 对象中主要存储：

```text
isa 指针
superclass 指针
属性信息
对象方法信息
协议信息
成员变量信息
```

### meta-class 对象

每个类在内存中有且只有一个 meta-class 对象。

meta-class 对象中主要存储：

```text
isa 指针
superclass 指针
类方法信息
```

---

# 二、isa 与 superclass

## 5. 对象的 isa 指针指向哪里？

### 答案

```text
instance 对象的 isa 指向 class 对象
class 对象的 isa 指向 meta-class 对象
meta-class 对象的 isa 指向基类的 meta-class 对象
```

例如：

```objc
MJPerson *person = [[MJPerson alloc] init];
Class cls = [MJPerson class];
Class metaCls = object_getClass(cls);
```

关系：

```text
person 的 isa -> MJPerson class
MJPerson class 的 isa -> MJPerson meta-class
MJPerson meta-class 的 isa -> NSObject meta-class
```

---

## 6. superclass 指针指向哪里？

### 答案

class 对象的 `superclass` 指向父类的 class 对象。

meta-class 对象的 `superclass` 指向父类的 meta-class 对象。

基类的 meta-class 的 `superclass` 指向基类的 class 对象。

例如：

```objc
@interface MJPerson : NSObject
@end

@interface MJStudent : MJPerson
@end
```

class 继承链：

```text
MJStudent class -> MJPerson class -> NSObject class -> nil
```

meta-class 继承链：

```text
MJStudent meta-class -> MJPerson meta-class -> NSObject meta-class -> NSObject class
```

---

## 7. 实例对象调用对象方法的过程是什么？

### 答案

```objc
MJStudent *stu = [[MJStudent alloc] init];
[stu run];
```

调用轨迹：

```text
stu instance
  -> isa 找到 MJStudent class
  -> 在 MJStudent class 的方法列表中查找 -run
  -> 找不到就通过 superclass 找 MJPerson class
  -> 继续向父类查找
  -> 找到方法实现后调用
```

对象方法存放在 class 对象中。

---

## 8. 类对象调用类方法的过程是什么？

### 答案

```objc
[MJStudent test];
```

调用轨迹：

```text
MJStudent class
  -> isa 找到 MJStudent meta-class
  -> 在 MJStudent meta-class 的方法列表中查找 +test
  -> 找不到就通过 superclass 找 MJPerson meta-class
  -> 继续向父类 meta-class 查找
  -> 找到方法实现后调用
```

类方法存放在 meta-class 对象中。

---

## 9. OC 的类信息存放在哪里？

### 答案

对象方法、属性、成员变量、协议信息存放在：

```text
class 对象中
```

类方法存放在：

```text
meta-class 对象中
```

成员变量的具体值存放在：

```text
instance 对象中
```

---

## 10. 为什么 64 位下 isa 需要位运算？

### 答案

64 位环境下，`isa` 不一定是单纯的类对象地址，而是经过优化的位域结构，常叫做 `nonpointer isa`。

它里面除了类对象地址，还可能保存：

```text
引用计数相关信息
是否有关联对象
是否使用 C++ 析构
是否正在释放
是否开启弱引用
```

所以需要通过掩码取出真实类地址：

```objc
isa & ISA_MASK
```

---

# 三、class / meta-class

## 11. class 对象和 meta-class 对象有什么区别？

### 答案

class 对象主要保存对象相关信息：

```text
对象方法
属性
成员变量
协议
superclass
```

meta-class 对象主要保存类相关信息：

```text
类方法
superclass
```

### 例子

```objc
- (void)run;
+ (void)test;
```

`-run` 存放在 class 对象中。

`+test` 存放在 meta-class 对象中。

---

## 12. 如何判断一个 Class 是否是 meta-class？

### 答案

可以使用 Runtime API：

```objc
BOOL result = class_isMetaClass(object_getClass([MJPerson class]));
```

示例：

```objc
MJPerson *person = [[MJPerson alloc] init];
Class cls = object_getClass(person);
Class metaCls = object_getClass([MJPerson class]);

NSLog(@"%d", class_isMetaClass(cls));      // 0
NSLog(@"%d", class_isMetaClass(metaCls));  // 1
```

---

# 四、isKindOfClass 与 isMemberOfClass

## 13. isKindOfClass 和 isMemberOfClass 的区别是什么？

### 答案

`isMemberOfClass:` 只判断对象是否正好是这个类的实例。

`isKindOfClass:` 判断对象是否是这个类或者它子类的实例。

### 示例

```objc
MJPerson *person = [[MJPerson alloc] init];
MJStudent *student = [[MJStudent alloc] init];

[person isMemberOfClass:[MJPerson class]];   // YES
[student isMemberOfClass:[MJPerson class]];  // NO
[student isMemberOfClass:[MJStudent class]]; // YES

[person isKindOfClass:[MJPerson class]];     // YES
[student isKindOfClass:[MJPerson class]];    // YES
[student isKindOfClass:[MJStudent class]];   // YES
```

### 底层理解

`isMemberOfClass:` 只比较：

```text
对象的 isa == 传入的 class
```

`isKindOfClass:` 会沿着继承链查找：

```text
对象的 isa -> class -> superclass -> superclass ...
```

---

## 14. 下面代码打印什么？

```objc
BOOL res1 = [[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [[MJPerson class] isKindOfClass:[MJPerson class]];
BOOL res4 = [[MJPerson class] isMemberOfClass:[MJPerson class]];

NSLog(@"%d %d %d %d", res1, res2, res3, res4);
```

### 答案

```text
1 0 0 0
```

### 解释

这里左边不是普通实例对象，而是类对象。

`[NSObject class]` 是 NSObject 类对象，它的 `isa` 指向 NSObject 元类。

NSObject 元类比较特殊，它的继承链最终能走到 NSObject 类对象，所以：

```objc
[[NSObject class] isKindOfClass:[NSObject class]]; // YES
```

`isMemberOfClass:` 只判断 `isa` 是否等于传入 class。

NSObject 类对象的 `isa` 是 NSObject 元类，不是 NSObject 类对象，所以：

```objc
[[NSObject class] isMemberOfClass:[NSObject class]]; // NO
```

MJPerson 类对象的 `isa` 是 MJPerson 元类，元类继承链是：

```text
MJPerson meta-class -> NSObject meta-class -> NSObject class
```

找不到 MJPerson class，所以后两个都是 NO。

---

# 五、KVO

## 15. KVO 的本质是什么？

### 答案

KVO 的本质是：

```text
Runtime 动态生成一个子类，并把当前对象的 isa 指向这个新子类。
```

例如：

```objc
MJPerson *person = [[MJPerson alloc] init];
```

添加 KVO 前：

```text
person 的 isa -> MJPerson class
```

添加 KVO 后：

```text
person 的 isa -> NSKVONotifying_MJPerson class
```

这个 `NSKVONotifying_MJPerson` 是系统动态生成的子类。

---

## 16. KVO 监听属性变化时，内部流程是什么？

### 答案

假设监听 `age`：

```objc
[person addObserver:self
         forKeyPath:@"age"
            options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
            context:nil];
```

修改：

```objc
person.age = 20;
```

实际流程：

```text
调用 NSKVONotifying_MJPerson 的 setAge:
  -> willChangeValueForKey:
  -> 调用父类原来的 setAge:
  -> didChangeValueForKey:
  -> didChangeValueForKey: 内部通知 observer
  -> observeValueForKeyPath:ofObject:change:context:
```

---

## 17. KVO 生成的子类中通常有哪些方法？

### 答案

系统动态生成的 KVO 子类一般会重写：

```text
setAge:
class
dealloc
_isKVOA
```

### 解释

`setAge:` 用于包裹 KVO 通知逻辑。

`class` 用于隐藏真实类型。虽然对象的 `isa` 指向 `NSKVONotifying_MJPerson`，但调用：

```objc
[person class]
```

返回的仍然是：

```text
MJPerson
```

---

## 18. 如何手动触发 KVO？

### 答案

手动调用：

```objc
willChangeValueForKey:
didChangeValueForKey:
```

示例：

```objc
[person willChangeValueForKey:@"age"];
person->_age = 20;
[person didChangeValueForKey:@"age"];
```

---

## 19. 直接修改成员变量会触发 KVO 吗？

### 答案

不会。

```objc
person->_age = 20;
```

这绕过了 setter，不会触发 KVO。

---

## 20. 通过 KVC 修改属性会触发 KVO 吗？

### 答案

会。

```objc
[person setValue:@20 forKey:@"age"];
```

KVC 会触发 KVO 通知。

---

# 六、KVC

## 21. KVC 是什么？

### 答案

KVC 全称是 Key-Value Coding，键值编码。

它允许通过字符串 key 访问对象属性。

常见 API：

```objc
- (void)setValue:(id)value forKey:(NSString *)key;
- (id)valueForKey:(NSString *)key;
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;
- (id)valueForKeyPath:(NSString *)keyPath;
```

---

## 22. setValue:forKey: 的赋值流程是什么？

### 答案

假设调用：

```objc
[person setValue:@20 forKey:@"age"];
```

KVC 会按下面流程查找。

第一步，查找 setter 方法：

```text
setAge:
_setAge:
```

找到就调用方法赋值。

第二步，如果没找到 setter，查看：

```objc
+ (BOOL)accessInstanceVariablesDirectly
```

默认返回 YES。

第三步，如果允许访问成员变量，按顺序查找：

```text
_age
_isAge
age
isAge
```

找到后直接赋值。

第四步，如果都找不到，调用：

```objc
setValue:forUndefinedKey:
```

然后抛出：

```text
NSUnknownKeyException
```

---

## 23. valueForKey: 的取值流程是什么？

### 答案

假设调用：

```objc
[person valueForKey:@"age"];
```

第一步，查找 getter 方法：

```text
getAge
age
isAge
_age
```

找到就调用方法取值。

第二步，如果没找到 getter，查看：

```objc
+ (BOOL)accessInstanceVariablesDirectly
```

默认返回 YES。

第三步，如果允许访问成员变量，按顺序查找：

```text
_age
_isAge
age
isAge
```

找到后直接取值。

第四步，如果都找不到，调用：

```objc
valueForUndefinedKey:
```

然后抛出：

```text
NSUnknownKeyException
```

---

## 24. accessInstanceVariablesDirectly 有什么作用？

### 答案

它决定 KVC 在找不到 setter/getter 时，是否可以直接访问成员变量。

默认返回 YES。

如果返回 NO，KVC 找不到方法时不会继续查找成员变量，而是直接抛异常。

---

# 七、Category

## 25. Category 的使用场景有哪些？

### 答案

Category 常用于：

```text
给已有类添加方法
拆分复杂类的功能模块
给系统类添加便捷方法
按业务功能组织代码
```

例如：

```objc
@interface NSString (MJExtension)
- (BOOL)mj_isBlank;
@end
```

Category 可以添加：

```text
对象方法
类方法
属性声明
协议
```

不能直接添加成员变量。

---

## 26. Category 的底层结构是什么？

### 答案

Category 编译后底层结构是：

```objc
struct category_t
```

里面存储：

```text
分类名
类名
对象方法列表
类方法列表
协议列表
属性列表
```

大致结构：

```objc
struct category_t {
    const char *name;
    classref_t cls;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    protocol_list_t *protocols;
    property_list_t *instanceProperties;
};
```

---

## 27. Category 的方法是如何合并到类中的？

### 答案

程序运行时，Runtime 会把 Category 的数据合并到类对象和元类对象中。

对象方法合并到 class 对象。

类方法合并到 meta-class 对象。

Category 的方法列表通常会插入到原类方法列表前面。

所以如果 Category 中的方法和原类方法同名，调用时通常会优先调用 Category 的方法。

注意：这不是严格意义上的覆盖，而是方法查找时先找到了 Category 的实现。

---

## 28. Category 和 Class Extension 有什么区别？

### 答案

| 对比项 | Category | Class Extension |
|---|---|---|
| 发生时机 | 运行时合并到类信息 | 编译时已经合并到类信息 |
| 是否可以添加方法 | 可以 | 可以 |
| 是否可以添加属性声明 | 可以 | 可以 |
| 是否可以添加成员变量 | 不可以 | 可以 |
| 主要用途 | 给已有类扩展功能 | 声明私有属性、私有方法、私有成员变量 |

核心区别：

```text
Class Extension 在编译期就成为类的一部分。
Category 在运行时才合并到类信息中。
```

---

## 29. Category 可以添加成员变量吗？

### 答案

不能直接添加成员变量。

原因是类的成员变量布局在编译时已经确定。

Category 是运行时才合并到类信息中的，此时对象内存布局已经固定，不能再增加成员变量。

但可以通过关联对象间接实现类似成员变量的效果。

---

# 八、+load 与 +initialize

## 30. Category 中可以有 +load 方法吗？

### 答案

可以。

```objc
@implementation MJPerson (Test)
+ (void)load {
    NSLog(@"Category load");
}
@end
```

`+load` 会在 Runtime 加载类和分类时调用。

---

## 31. +load 什么时候调用？

### 答案

`+load` 在 Runtime 加载类、分类时调用。

特点：

```text
程序启动阶段调用
每个类、每个分类的 +load 只调用一次
不需要手动发送消息
```

---

## 32. +load 的调用顺序是什么？

### 答案

整体顺序：

```text
先调用类的 +load
再调用分类的 +load
```

类之间：

```text
先父类，后子类
```

分类之间：

```text
按照编译顺序调用
```

---

## 33. +load 是通过 objc_msgSend 调用的吗？

### 答案

不是。

`+load` 是根据函数地址直接调用的，不是通过 `objc_msgSend`。

所以类的 `+load` 和分类的 `+load` 都会被调用，不会因为同名而只调用一个。

---

## 34. +initialize 方法什么时候调用？

### 答案

`+initialize` 会在类第一次接收到消息时调用。

例如：

```objc
[MJPerson alloc];
[MJPerson class];
```

类第一次收到消息时，Runtime 会触发 `+initialize`。

---

## 35. +initialize 的调用顺序是什么？

### 答案

```text
先父类，后子类
```

第一次使用子类时，会先保证父类完成初始化。

---

## 36. +load 和 +initialize 的区别是什么？

### 答案

| 对比项 | +load | +initialize |
|---|---|---|
| 调用时机 | Runtime 加载类、分类时 | 类第一次接收到消息时 |
| 调用方式 | 直接函数地址调用 | 通过 objc_msgSend 调用 |
| 调用次数 | 每个类、分类各一次 | 每个类通常一次 |
| 分类影响 | 类和分类都调用 | 分类可能覆盖原类实现 |
| 继承影响 | 先父类后子类 | 先父类后子类 |
| 是否懒加载 | 不是 | 是 |

---

## 37. 为什么父类的 +initialize 可能被调用多次？

### 答案

因为 `+initialize` 是通过 `objc_msgSend` 调用的。

如果子类没有实现 `+initialize`，会沿着继承链找到父类的实现。

示例：

```objc
@implementation MJPerson
+ (void)initialize {
    NSLog(@"MJPerson initialize");
}
@end

@implementation MJStudent
@end
```

第一次使用 `MJPerson` 会调用一次。

第一次使用 `MJStudent` 时，由于 `MJStudent` 没有实现，会再次调用父类的实现。

防止方式：

```objc
+ (void)initialize {
    if (self == [MJPerson class]) {
        NSLog(@"MJPerson initialize");
    }
}
```

---

# 九、关联对象

## 38. 如何给 Category 间接添加成员变量？

### 答案

通过关联对象。

常用 API：

```objc
void objc_setAssociatedObject(id object,
                              const void *key,
                              id value,
                              objc_AssociationPolicy policy);

id objc_getAssociatedObject(id object,
                            const void *key);

void objc_removeAssociatedObjects(id object);
```

示例：

```objc
#import <objc/runtime.h>

@implementation MJPerson (Extension)

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self,
                             @selector(name),
                             name,
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, @selector(name));
}

@end
```

---

## 39. 关联对象常见 key 写法有哪些？

### 答案

静态指针：

```objc
static void *MJNameKey = &MJNameKey;
```

静态 char：

```objc
static char MJNameKey;
```

字符串：

```objc
@"name"
```

getter 的 selector：

```objc
@selector(name)
```

推荐使用 `@selector(name)`，简单且不容易冲突。

---

## 40. 关联对象的策略有哪些？

### 答案

| 策略 | 对应属性修饰符 |
|---|---|
| OBJC_ASSOCIATION_ASSIGN | assign |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC | strong, nonatomic |
| OBJC_ASSOCIATION_COPY_NONATOMIC | copy, nonatomic |
| OBJC_ASSOCIATION_RETAIN | strong, atomic |
| OBJC_ASSOCIATION_COPY | copy, atomic |

---

## 41. 关联对象存在哪里？

### 答案

关联对象不是存储在被关联对象本身内存中。

它存储在 Runtime 维护的全局关联对象表中。

可以理解为：

```text
全局 Map {
    对象地址 : {
        key : association
    }
}
```

所以 Category 并没有真的改变对象内存布局。

---

# 十、Block

## 42. Block 的本质是什么？

### 答案

Block 本质上是一个 OC 对象。

它内部也有 `isa` 指针。

Block 封装了：

```text
函数调用
函数调用环境
捕获的变量
```

底层结构大致包含：

```objc
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```

---

## 43. Block 有哪几种类型？

### 答案

Block 有 3 种类型：

```text
__NSGlobalBlock__
__NSStackBlock__
__NSMallocBlock__
```

它们最终都继承自 `NSBlock`。

---

## 44. 什么情况下是全局 Block？

### 答案

没有访问 auto 局部变量的 Block 是全局 Block。

```objc
void (^block)(void) = ^{
    NSLog(@"hello");
};
```

类型通常是：

```text
__NSGlobalBlock__
```

---

## 45. 什么情况下是栈 Block？

### 答案

访问了 auto 局部变量，并且没有 copy 的 Block 是栈 Block。

```objc
int age = 10;
void (^block)(void) = ^{
    NSLog(@"%d", age);
};
```

MRC 下初始通常是：

```text
__NSStackBlock__
```

---

## 46. 什么情况下是堆 Block？

### 答案

栈 Block 调用 copy 后，会变成堆 Block。

```objc
int age = 10;
void (^block)(void) = [^{
    NSLog(@"%d", age);
} copy];
```

类型是：

```text
__NSMallocBlock__
```

---

## 47. Block 调用 copy 后会发生什么？

### 答案

| 原类型 | copy 后 |
|---|---|
| __NSGlobalBlock__ | 还是自己 |
| __NSStackBlock__ | 复制到堆，变成 __NSMallocBlock__ |
| __NSMallocBlock__ | 引用计数 +1 |

---

## 48. 为什么 Block 属性建议用 copy？

### 答案

因为 Block 默认可能在栈上。

如果栈上的 Block 超出作用域后还被使用，可能造成野指针崩溃。

使用 `copy` 可以把栈 Block 复制到堆上，延长生命周期。

推荐：

```objc
@property (nonatomic, copy) void (^block)(void);
```

ARC 下使用 strong 很多时候也可以正常工作，但语义上仍推荐 copy。

---

## 49. ARC 下什么时候 Block 会自动 copy 到堆上？

### 答案

ARC 下，以下情况编译器会自动把栈 Block copy 到堆上：

```text
Block 作为函数返回值
Block 赋值给 __strong 指针
Block 作为 Cocoa API 中 usingBlock 方法参数
Block 作为 GCD API 参数
```

---

# 十一、Block 变量捕获

## 50. Block 的变量捕获规则是什么？

### 答案

| 变量类型 | 是否捕获到 Block 内部 | 访问方式 |
|---|---|---|
| auto 局部变量 | 会 | 值传递 |
| static 局部变量 | 会 | 指针传递 |
| 全局变量 | 不会 | 直接访问 |

---

## 51. Block 访问 auto 局部变量时，是值传递还是指针传递？

### 答案

值传递。

```objc
int age = 10;

void (^block)(void) = ^{
    NSLog(@"%d", age);
};

age = 20;
block();
```

打印：

```text
10
```

Block 创建时捕获的是 age 当时的值。

---

## 52. Block 访问 static 局部变量时，是值传递还是指针传递？

### 答案

指针传递。

```objc
static int age = 10;

void (^block)(void) = ^{
    NSLog(@"%d", age);
};

age = 20;
block();
```

打印：

```text
20
```

---

## 53. Block 访问全局变量时，会捕获吗？

### 答案

不会捕获，直接访问。

```objc
int age = 10;

void (^block)(void) = ^{
    NSLog(@"%d", age);
};

age = 20;
block();
```

打印：

```text
20
```

---

## 54. Block 内部为什么不能直接修改 auto 变量？

### 答案

因为 auto 局部变量是值传递捕获到 Block 内部的。

```objc
int age = 10;

void (^block)(void) = ^{
    age = 20; // 报错
};
```

解决方式：使用 `__block`。

---

## 55. Block 修改 NSMutableArray 需要加 __block 吗？

### 答案

不需要。

```objc
NSMutableArray *array = [NSMutableArray array];

void (^block)(void) = ^{
    [array addObject:@"123"];
};
```

这里没有修改 `array` 这个指针变量本身，只是修改它指向的对象内部内容。

下面这种才需要 `__block`：

```objc
array = [NSMutableArray array];
```

---

# 十二、__block

## 56. __block 的作用是什么？

### 答案

`__block` 可以让 Block 内部修改 auto 局部变量。

```objc
__block int age = 10;

void (^block)(void) = ^{
    age = 20;
};

block();
NSLog(@"%d", age); // 20
```

---

## 57. __block 可以修饰哪些变量？

### 答案

`__block` 可以修饰 auto 局部变量。

不能修饰：

```text
全局变量
static 静态变量
```

因为全局变量和 static 变量本来就可以在 Block 内部直接修改。

---

## 58. __block 的底层原理是什么？

### 答案

编译器会把 `__block` 变量包装成一个结构体对象。

例如：

```objc
__block int age = 10;
```

底层大致变成：

```objc
struct __Block_byref_age_0 {
    void *__isa;
    struct __Block_byref_age_0 *__forwarding;
    int __flags;
    int __size;
    int age;
};
```

Block 内部访问的是：

```objc
age->__forwarding->age
```

---

## 59. __block 变量为什么有 __forwarding 指针？

### 答案

因为 `__block` 变量可能从栈复制到堆。

`__forwarding` 用来保证不管变量在栈上还是堆上，都能访问到正确的那份变量。

Block 在栈上时：

```text
__forwarding 指向自己
```

Block copy 到堆上后：

```text
栈上的 __block 变量的 __forwarding 指向堆上的 __block 变量
堆上的 __block 变量的 __forwarding 指向自己
```

---

## 60. Block 在栈上时，会强引用捕获的对象吗？

### 答案

不会。

当 Block 在栈上时，对捕获的对象类型 auto 变量不会产生强引用。

---

## 61. Block 被 copy 到堆上时，会强引用捕获的对象吗？

### 答案

会根据变量修饰符决定。

如果是 `__strong`，会强引用。

如果是 `__weak`，会弱引用。

如果是 `__unsafe_unretained`，不会强引用，也不会自动置 nil。

Block copy 到堆上时，会调用内部 copy 函数，copy 函数内部调用：

```objc
_Block_object_assign
```

然后根据变量修饰符处理引用关系。

---

## 62. Block 从堆上销毁时会发生什么？

### 答案

会调用 Block 内部的 dispose 函数。

里面会调用：

```objc
_Block_object_dispose
```

释放捕获的对象或 `__block` 变量。

---

## 63. __block 修饰对象时，ARC 和 MRC 下有什么区别？

### 答案

ARC 下：

```objc
__block NSObject *obj = [[NSObject alloc] init];
```

默认仍然是强引用。

MRC 下：

```text
__block 修饰对象不会 retain 对象
```

所以 MRC 下可以用 `__block` 解决循环引用。

---

# 十三、循环引用

## 64. Block 为什么容易造成循环引用？

### 答案

常见场景：

```objc
@interface MJPerson : NSObject
@property (nonatomic, copy) void (^block)(void);
@end
```

```objc
self.block = ^{
    NSLog(@"%@", self);
};
```

引用关系：

```text
self 强引用 block
block 强引用 self
```

形成：

```text
self -> block -> self
```

对象无法释放。

---

## 65. ARC 下如何解决 Block 循环引用？

### 答案

常用 `__weak`：

```objc
__weak typeof(self) weakSelf = self;

self.block = ^{
    NSLog(@"%@", weakSelf);
};
```

更安全的写法：

```objc
__weak typeof(self) weakSelf = self;

self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;

    [strongSelf doSomething];
};
```

---

## 66. 为什么常用 weak-strong dance？

### 答案

```objc
__weak typeof(self) weakSelf = self;

self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;

    [strongSelf doSomething];
};
```

`weakSelf` 避免循环引用。

`strongSelf` 保证 Block 执行过程中对象不会突然释放。

---

## 67. __unsafe_unretained 可以解决循环引用吗？

### 答案

可以避免强引用，但不推荐。

因为对象释放后，`__unsafe_unretained` 不会自动置 nil。

继续访问会产生野指针崩溃。

ARC 下推荐使用 `__weak`。

---

## 68. ARC 下能用 __block 解决循环引用吗？

### 答案

一般不推荐。

ARC 下 `__block` 默认仍然会强引用对象。

如果要用，需要在 Block 内部手动置空：

```objc
__block MJPerson *person = self;

self.block = ^{
    NSLog(@"%@", person);
    person = nil;
};
```

缺点：必须保证 Block 被调用，否则循环引用仍然存在。

---

## 69. MRC 下如何解决 Block 循环引用？

### 答案

MRC 下常用：

```text
__block
__unsafe_unretained
```

因为 MRC 下 `__block` 修饰对象不会 retain 对象。

---

# 十四、代码打印题

## 70. self 和 super 打印题

### 代码

```objc
@interface MJPerson : NSObject
@end

@interface MJStudent : MJPerson
@end

@implementation MJStudent

- (instancetype)init {
    if (self = [super init]) {
        NSLog(@"[self class] = %@", [self class]);
        NSLog(@"[super class] = %@", [super class]);
        NSLog(@"[self superclass] = %@", [self superclass]);
        NSLog(@"[super superclass] = %@", [super superclass]);
    }
    return self;
}

@end
```

执行：

```objc
[[MJStudent alloc] init];
```

### 答案

```text
[self class] = MJStudent
[super class] = MJStudent
[self superclass] = MJPerson
[super superclass] = MJPerson
```

### 解释

`super` 不是父类对象。

`super` 只是告诉编译器：从父类开始查找方法实现。

但是消息接收者仍然是 `self`。

所以 `[super class]` 的接收者还是当前 `MJStudent` 实例，只是从父类开始查找 `class` 方法，最终返回仍然是 `MJStudent`。

---

## 71. 伪造对象打印题

### 代码

```objc
@interface MJPerson : NSObject

@property (nonatomic, copy) NSString *name;

- (void)print;

@end

@implementation MJPerson

- (void)print {
    NSLog(@"my name's %@", self.name);
}

@end
```

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    id cls = [MJPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
```

### 答案

可以执行，但结果不稳定。

常见可能打印：

```text
my name's <ViewController: 0x...>
```

也可能打印：

```text
my name's (null)
```

甚至可能崩溃。

### 解释

关键代码：

```objc
id cls = [MJPerson class];
void *obj = &cls;
```

`obj` 指向局部变量 `cls` 的地址。

而 `cls` 变量里面存的是 `[MJPerson class]`。

执行：

```objc
[(__bridge id)obj print];
```

Runtime 会把 `obj` 当成一个对象。

对象内存的第一块区域会被当成 `isa`。

刚好 `obj` 指向的内存里放着 `MJPerson class`，于是 Runtime 认为：

```text
这个假对象的 isa 是 MJPerson class
```

所以可以找到 `print` 方法。

但是 `self.name` 会去读取对象的成员变量。

这个对象是伪造的，它后面的内存其实是栈上的其他内容，所以读出来的值不确定。

标准结论：

```text
这段代码属于伪造对象，能不能正常打印依赖栈内存布局，属于未定义行为。
```

---

# 十五、快速背诵版

## OC 对象

```text
instance：isa + 成员变量值
class：isa + superclass + 属性 + 对象方法 + 协议 + 成员变量描述
meta-class：isa + superclass + 类方法
```

## isa

```text
instance isa -> class
class isa -> meta-class
meta-class isa -> 基类 meta-class
```

## superclass

```text
class superclass -> 父类 class
meta-class superclass -> 父类 meta-class
基类 meta-class superclass -> 基类 class
```

## KVO

```text
Runtime 动态生成 NSKVONotifying_XXX 子类
修改对象 isa 指向这个子类
重写 setter
setter 中调用 willChange、原 setter、didChange
```

## KVC 赋值

```text
setKey:
_setKey:
accessInstanceVariablesDirectly
_key
_isKey
key
isKey
setValue:forUndefinedKey:
```

## KVC 取值

```text
getKey
key
isKey
_key
accessInstanceVariablesDirectly
_key
_isKey
key
isKey
valueForUndefinedKey:
```

## Category

```text
底层是 category_t
运行时合并到类信息
对象方法合并到 class
类方法合并到 meta-class
不能直接添加成员变量
可以用关联对象间接实现
```

## +load

```text
Runtime 加载类、分类时调用
先类后分类
类：先父类后子类
分类：按编译顺序
直接函数地址调用，不走 objc_msgSend
```

## +initialize

```text
类第一次收到消息时调用
先父类后子类
走 objc_msgSend
分类可能覆盖原类 initialize
子类没实现时，可能调用父类 initialize
```

## Block

```text
Block 本质是 OC 对象
封装函数调用和调用环境
有 isa
```

## Block 类型

```text
__NSGlobalBlock__：没有访问 auto 变量
__NSStackBlock__：访问 auto 变量，未 copy
__NSMallocBlock__：栈 Block copy 到堆
```

## 变量捕获

```text
auto 局部变量：值传递
static 局部变量：指针传递
全局变量：不捕获，直接访问
```

## __block

```text
让 Block 内部可以修改 auto 变量
底层包装成 __Block_byref_xxx 结构体
通过 __forwarding 保证访问正确变量
```

## 循环引用

```text
self -> block -> self
```

ARC 推荐写法：

```objc
__weak typeof(self) weakSelf = self;

self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;

    [strongSelf doSomething];
};
```
