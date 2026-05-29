# iOS 知识点汇总

> 适用场景：iOS 面试复习、知识查漏补缺、专题复盘。
> 排版原则：按“底层机制 -> 语言特性 -> 并发通信 -> UI 渲染 -> 工程能力 -> 计算机基础”组织；统一为书稿式层级，去除资料拼接痕迹。

## 阅读方法

- **先建立主线**：Runtime、RunLoop、内存管理、多线程是 iOS 面试的底层基础。
- **再补高频场景**：Block、通知、视图渲染、性能优化对应常见工程追问。
- **最后串联表达**：组件化、设计模式、算法与综合题用于训练系统化回答。
- **答题结构**：每个问题建议按“概念 -> 原理 -> 场景 -> 风险点 -> 面试表达”复盘。

## 知识体系

| 层级 | 主题 | 重点能力 |
| --- | --- | --- |
| 底层机制 | Runtime、RunLoop、KVO | 消息发送、对象模型、事件循环、观察机制 |
| 内存与语言 | 内存管理、Block、weak、ARC | 生命周期、引用关系、拷贝语义、循环引用 |
| 并发通信 | 多线程、NSNotification | GCD、NSOperation、线程安全、同步/异步通知 |
| UI 与渲染 | 视图与图形、事件响应、离屏渲染 | 响应链、布局、绘制、流畅度优化 |
| 工程能力 | 性能优化、组件化、设计模式 | 启动优化、包体优化、解耦、模块治理 |
| 计算机基础 | 数据结构、算法、网络、数据库 | 复杂度、排序、链表、二叉树、HTTPS |

## 章节目录

1. [第一章 Runtime](#第一章runtime)
2. [第二章 RunLoop 与 KVO](#第二章runloop-与-kvo)
3. [第三章 内存管理](#第三章内存管理)
4. [第四章 多线程](#第四章多线程)
5. [第五章 Block](#第五章block)
6. [第六章 通知机制（NSNotification）](#第六章通知机制nsnotification)
7. [第七章 视图与图形](#第七章视图与图形)
8. [第八章 数据结构](#第八章数据结构)
9. [第九章 算法](#第九章算法)
10. [第十章 性能优化](#第十章性能优化)
11. [第十一章 组件化](#第十一章组件化)
12. [第十二章 综合面试题与扩展专题](#第十二章综合面试题与扩展专题)

---

<a id="第一章runtime"></a>
## 第一章 Runtime


### 1.1 Category 的实现原理？

- Category 在底层对应的是 `category_t` 结构体。在运行时，分类的方法会被以倒序插入到原有方法列表的最前面，因此当不同分类添加了同名方法时，实际生效的是最后参与编译的那个分类的方法。

- Category 在刚编译完成时，和原来的类是分开的；只有在程序运行起来后，才会通过 Runtime 把 Category 与原来的类合并到一起。

### 1.2 isa指针的理解，对象的isa指针指向哪里？isa指针有哪两种类型？

- isa 等价于 is kind of

    实例对象的 isa 指向类对象

    类对象的 isa 指向元类对象

    元类对象的 isa 指向根元类（即 NSObject 的元类）；根元类的 isa 则指向根类 NSObject 本身，从而形成闭环

- isa 有两种类型

    纯指针，指向内存地址

    NON_POINTER_ISA，除了内存地址，还存有一些其他信息

### 1.3 Objective-C 如何实现多重继承？

Object-c的类没有多继承,只支持单继承,如果要实现多继承的话，可使用如下几种方式间接实现

- 通过组合实现

    A和B组合，作为C类的组件

- 通过协议实现

    C类实现A和B类的协议方法

- 消息转发实现

    forwardInvocation:方法

### 1.4 runtime 如何实现 weak 属性？

weak 此特质表明该属性定义了一种「非拥有关系」(nonowning relationship)。为这种属性设置新值时，设置方法既不持有新值（新指向的对象），也不释放旧值（原来指向的对象）。

runtime 对注册的类，会进行内存布局，从一个粗粒度的概念上来讲，这时候会有一个 hash 表，这是一个全局表，表中是用 weak 指向的对象内存地址作为 key，用所有指向该对象的 weak 指针表作为 value。当此对象的引用计数为 0 的时候会 dealloc，假如该对象内存地址是 a，那么就会以 a 为 key，在这个 weak 表中搜索，找到所有以 a 为键的 weak 对象，从而设置为 nil。

runtime 如何实现 weak 属性具体流程大致分为 3 步：

1. 初始化时：runtime 会调用 objc_initWeak 函数，初始化一个新的 weak 指针指向对象的地址。

2. 添加引用时：objc_initWeak 函数会调用 objc_storeWeak() 函数，objc_storeWeak() 的作用是更新指针指向（指针可能原来指向着其他对象，这时候需要将该 weak 指针与旧对象解除绑定，会调用到 weak_unregister_no_lock），如果指针指向的新对象非空，则创建对应的弱引用表，将 weak 指针与新对象进行绑定，会调用到 weak_register_no_lock。在这个过程中，为了防止多线程中竞争冲突，会有一些锁的操作。

3. 释放时：调用 clearDeallocating 函数，clearDeallocating 函数首先根据对象地址获取所有 weak 指针地址的数组，然后遍历这个数组把其中的数据设为 nil，最后把这个 entry 从 weak 表中删除，最后清理对象的记录。

### 1.5 讲一下 OC 的消息机制

- OC中的方法调用其实都是转成了objc_msgSend函数的调用，给receiver（方法调用者）发送了一条消息（selector方法名）

- objc_msgSend底层有3大阶段，消息发送（当前类、父类中查找）、动态方法解析、消息转发

### 1.6 runtime具体应用

- 利用关联对象（AssociatedObject）给分类添加属性

- 遍历类的所有成员变量（修改textfield的占位文字颜色、字典转模型、自动归档解档）

- 交换方法实现（交换系统的方法）

- 利用消息转发机制解决方法找不到的异常问题

- KVC 字典转模型

### 1.7 runtime如何通过selector找到对应的IMP地址？

每一个类对象中都有一个实例方法列表（以及方法缓存 cache）。

- 类方法则存放在类对象的 isa 指针所指向的元类对象中。

- 方法列表中每个方法结构体中记录着方法的名称,方法实现,以及参数类型，其实selector本质就是方法名称,通过这个方法名称就可以在方法列表中找到对应的方法实现。

- 当我们发送一个消息给一个NSObject对象时，这条消息会在对象的类对象方法列表里查找。

- 当我们发送一个消息给一个类时，这条消息会在类的Meta Class对象的方法列表里查找。

### 1.8 简述下Objective-C中调用方法的过程

Objective-C是动态语言，每个方法在运行时会被动态转为消息发送，即：objc_msgSend(receiver, selector)，整个过程介绍如下：

- objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类

- 然后在该类中的方法列表以及其父类方法列表中寻找方法运行

- 如果，在最顶层的父类（一般也就NSObject）中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX

- 但是在这之前，objc的运行时会给出三次拯救程序崩溃的机会，这三次拯救程序崩溃的说明见问题《什么时候会报unrecognized selector的异常》中的说明。

### 1.9 load和initialize的区别

两者都由运行时自动调用、无需手动调用 super，且默认仅调用一次（主动显式调用除外）。

- `+load` 在程序启动、类被装载到运行时系统时调用，早于 main 函数；`+initialize` 则在类（或其子类）第一次收到消息时才被惰性调用，通常发生在 main 函数之后。两者均由运行时自动调用，不应手动调用。

- `+initialize` 通过消息机制调用，子类即使没有实现也会沿用父类的实现（因而父类的 `+initialize` 可能被多次调用）；而 `+load` 不走消息机制，运行时会对每个实现了 `+load` 的类和分类各自独立调用一次，子类未实现时并不会"沿用"父类的 `+load`。

- load方法通常用来进行Method Swizzle，initialize方法一般用于初始化全局变量或静态变量。

- load和initialize方法由运行时加锁调用，因此它们本身是线程安全的。实现时应尽量保持简单，避免阻塞线程，也不要在其中额外加锁，以免与运行时持有的锁相互等待而死锁。

### 1.10 怎么理解Objective-C是动态运行时语言。

- 其核心是将数据类型的确定由编译时推迟到运行时。这一问题主要涉及两个概念：运行时与多态。

- 简单来说，运行时机制让我们能够在程序运行时才决定一个对象的类型，以及调用该对象所属类型的指定方法。

- 多态：不同对象以各自的方式响应相同消息的能力，称为多态。

- 例如，假设生物类（Life）都拥有一个相同的方法 eat；人类属于生物，猪也属于生物，二者都继承 Life 后各自实现自己的 eat，而调用时我们只需统一调用各自的 eat 方法即可。也就是不同对象以各自的方式响应了相同的消息（响应了 eat 这个选择器）。因此可以说，运行时机制是多态的基础。


### 1.11 Runtime 结构模型与消息机制

[objc-runtime源码地址](https://github.com/RetVal/objc-runtime)
[objc4官方源码地址](https://opensource.apple.com/tarballs/objc4/)

#### 1.11.1 结构模型

#### 1.11.2 Runtime 内存模型

#### 1.11.3 对象

OC中的对象指向的是一个`objc_object`指针类型，`typedef struct objc_object *id;`从它的结构体中可以看出，它包括一个isa指针，指向的是这个对象的类对象,一个对象实例就是通过这个isa找到它自己的Class，而这个Class中存储的就是这个实例的方法列表、属性列表、成员变量列表等相关信息的。

```
/// Represents an instance of a class.
struct objc_object {
 Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

这个objc_object 的实现比较长 在这里[查看](https://github.com/RetVal/objc-runtime/blob/master/runtime/objc-private.h)

#### 1.11.4 类

在OC中的类是用Class来表示的，实际上它指向的是一个`objc_class`的指针类型，`typedef struct objc_class *Class;`
对应的结构体如下：

```
struct objc_class {
 Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
 Class _Nullable super_class                              OBJC2_UNAVAILABLE;
 const char * _Nonnull name                               OBJC2_UNAVAILABLE;
 long version                                             OBJC2_UNAVAILABLE;
 long info                                                OBJC2_UNAVAILABLE;
 long instance_size                                       OBJC2_UNAVAILABLE;
 struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
 struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
 struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
 struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

}
```

#### 1.11.5 class 与 object 小结

从结构体中定义的变量可知，OC的`Class`类型包括如下

数据（即：元数据`metadata`）：`super_class`（父类类对象）;
name（类对象的名称）;
version、info（版本和相关信息）;
instance_size（实例内存大小）;
ivars（实例变量列表）；
methodLists（方法列表）；
cache（缓存）；
protocols（实现的协议列表）;
当然也包括一个isa指针，这说明Class也是一个对象类型，所以我们称之为类对象， 这里的isa指向的是元类对象（metaclass），元类中保存了创建类对象（Class）的类方法的全部信息。

[![Objective-C的对象原型继承链](https://upload-images.jianshu.io/upload_images/13277235-36a257745b6ff11c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/class_inherit.png)

从图中可知，最终的基类`NSObject`的元类对象`isa`指向的是自己本身，从而形成一个闭环。
元类（`Meta Class`）：是一个类对象的类，即：Class的类，这里保存了类方法等相关信息。
我们再看一下类对象中存储的方法、属性、成员变量等信息的结构体
`objc_ivar_list`：存储了类的成员变量，
可以通过`object_getIvar`或`class_copyIvarList`获取；
另外这两个方法是用来获取类的属性列表的`class_getProperty`和`class_copyPropertyList`，属性和成员变量是有区别的。

```
struct objc_ivar {
 char * _Nullable ivar_name                               OBJC2_UNAVAILABLE;
 char * _Nullable ivar_type                               OBJC2_UNAVAILABLE;
 int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
 int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;

struct objc_ivar_list {
 int ivar_count                                           OBJC2_UNAVAILABLE;
#ifdef __LP64__
 int space                                                OBJC2_UNAVAILABLE;
#endif
 /* variable length structure */
 struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;
}
```

`objc_method_list`：存储了类的方法列表，可以通过`class_copyMethodList`获取。

结构体如下:

```
struct objc_method {
 SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
 char * _Nullable method_types                            OBJC2_UNAVAILABLE;
 IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

struct objc_method_list {
 struct objc_method_list * _Nullable obsolete             OBJC2_UNAVAILABLE;

 int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
 int space                                                OBJC2_UNAVAILABLE;
#endif
 /* variable length structure */
 struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}
```

`objc_protocol_list`：储存了类的协议列表，可以通过`class_copyProtocolList`获取。

结构体如下：

```
struct objc_protocol_list {
 struct objc_protocol_list * _Nullable next;
 long count;
 __unsafe_unretained Protocol * _Nullable list[1];
};
```
此问题参考[介绍下runtime的内存模型（isa、对象、类、metaclass、结构体的存储信息等）](https://developer.aliyun.com/ask/282811)

#### 1.11.6 为什么要设计metaclass?

先说结论：metaclass 的设计是为了**复用消息传递机制**，它本质上只是为实现这一复用目的而引入的工具。在 Objective-C 中，**每个类都拥有自己独立的元类**，类对象的 isa 指向其元类，正是这套 isa 链让类方法的查找可以复用与实例方法相同的消息传递流程。由于 Objective-C 的语言特性基本借鉴自 Smalltalk，而 MetaClass 的设计是 Smalltalk-80 引入的，因此 Objective-C 也就有了 metaclass 的设计。

> 本质上因为Smalltalk的面向对象的亮点是它的**消息发送机制**.

回答这个问题之前我们先回看一下上边的Objective-C的对象原型继承链[![Objective-C的对象原型继承链](https://upload-images.jianshu.io/upload_images/13277235-d7fd4cd6cffceb0f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/class_inherit2.jpg)

通过上图我们明白如下 重点内容:

- **实例的实例方法函数存在类结构体中**
- **类方法函数存在metaclass结构体中**

而Objective-C的方法调用（消息）就会根据对象去找isa指针指向的Class对象中的方法列表找到对应的方法。 > isa 指向的类就是我们创建实例的类型.

需要澄清一个常见误区：在 Objective-C 中，**并不是所有类共用同一个元类，而是每个类都拥有各自独立的元类**。这一点与早期 Smalltalk-76（所有类的 isa 都指向同一个特殊类）不同，正是 Smalltalk-80 起为每个类引入独立 MetaClass 的设计。

#### 1.11.7 Smalltalk中的metaclass

Smalltalk，被公认为历史上第二个面向对象的语言，其亮点是它的**消息发送机制**。
Smalltalk中的MetaClass的设计是Smalltalk-80加入的。而之前的Smalltalk-76，并不是每个类有一个MetaClass，而是所有类的isa指针都指向一个特殊的类，叫做Class(这种设计之后也被Java借鉴了）。
而每个类都有自己MetaClass的设计，加入的原因是，因为Smalltalk里面，类是对象，而对象就可以响应消息，那么类的消息的响应的方法就应该由类的类去存储，而每个MetaClass就持有每个类的类方法。

#### 1.11.8 每个MetaClass的isa指针指向什么？

如果MetaClass再有MetaClass，那么这个关系将无穷无尽。Smalltalk里的解决方案是，指向同一个叫MetaClass的类。

#### 1.11.9 MetaClass的isa指针指向什么？

指向他的实例，也就是实例的isa指向MetaClass，同时MetaClassisa指向实例，相互指着。

那么Smalltalk的继承关系，其实和Objective-C的很像了（后面有class的是前者的MetaClass）。

[![](https://upload-images.jianshu.io/upload_images/13277235-5ea051d1aa043b59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/class_inherit2_smaltalk.png)

#### 1.11.10 类方法存放位置的设计问题

这个问题，我思索许久，发现其实是一个对面向对象的哲学思想问题，要对这个问题下结论，不得不重新讲讲面向对象

#### 1.11.11 从Smalltalk重新认识面向对象

以前谈到面向对象，总会提到，面向对象三特征：封装、继承、多态。但其实，面向对象中也分流派，如C++这种来自Simula的设计思想的，更注重的是类的划分，因为方法调用是静态的。而如Objective-C这种借鉴Smalltalk的，更注重的是消息传递，是动态响应消息。

而面向对象三种特征，更基于的是类的划分而提出的。

这两种思想最大的不同，我认为是自上而下和自下而上的思考方式。

- 类的划分，要求类的设计者是以一个很高的层次去设计这个类，提取出类的特性和本质，进行类的构建。知道类型才可以去发送消息给对象。
- 消息传递，要求的是类的设计者以消息为起点去构建类，也就是对外界的变化进行响应，而不关心自身的类型，设计接口。尝试理解消息，无法处理则进行特殊处理。 在此不讨论两种方式的优劣之分，而着重讲讲Smalltalk这种设计。

消息传递对于面向对象的设计，其实在于给出一种对消息的解决方案。而面向对象优点之一的复用，在这种设计里，更多在于复用解决方案，而不是单纯的类本身。这种思想就如设计组件一般，关心接口，关心组合而非类本身。其实之所以有MetaClass这种设计，我的理解并不是先有MetaClass，而是在万物都是对象的Smalltalk里，向对象发送消息的基本解决方案是统一的，希望复用的。而实例和类之间用的这一套通过isa指针指向的Class单例中存储方法列表和查询方法的解决方案的流程，是应该在类上复用的，而MetaClass就顺理成章出现罢了。

#### 1.11.12 MetaClass 设计小结

#### 1.11.13 MetaClass 与类方法的关系

我的理解是，可以，但不Smalltalk。这样的设计是C++那种自上而下的设计方式，类方法也是类的一种特征描述。而Smalltalk的精髓正在于消息传递，复用消息传递才是根本目的，而MetaClass只不过是因此需要的一个工具罢了。

参考[为什么Objective-C中有MetaClass这个设计？](https://www.jianshu.com/p/c1793bc2ca13)

#### 1.11.14 **class_copyIvarList()** & **class_copyPropertyList()**区别

先说结论:

- **class_copyIvarList()** 能获取到所有的成员变量,包括 花括号内的变量(`.h`和`.m`都包括).
- **class_copyPropertyList()** 只能获取到 以`@property`关键字 声明的中属性(`.h`和`.m`都包括)

区别:

- `class_copyIvarList()`获取默认是带下划线的变量
- `class_copyPropertyList()`获取默认是不带下划线的变量名称.

> 但是以上两个方法都只能获取到当前类的属性和变量（也就是说获取不到父类的属性和变量）

---

举例说明:

我们声明一个`ClassA` 通过 调试代码实现

```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface ClassA : NSObject {
 int _a;
 int _b;
 int _c;
 CGFloat d; //不推荐这样写
}

@property (nonatomic, strong) NSArray          *arrayA;
@property (nonatomic, copy  ) NSString         *stringA;
@property (nonatomic, assign) dispatch_queue_t testQueue;

@end

@implementation ClassA
@end
```

如果是通过`class_copyIvarList()`函数获取则打印如下结果.

```
--- class_copyIvarList ↓↓↓---
_a
_b
_c
d
_arrayA
_stringA
_testQueue
--------------END----------------
```

如果是通过`class_copyPropertyList()`函数获取则打印如下结果.

```
--- class_copyPropertyList ↓↓↓---
arrayA
stringA
testQueue
--------------END----------------
```

debug代码如下:

```
- (void)printIvarOrProperty {
 NSLog(@"--- class_copyPropertyList ↓↓↓---");
 ClassA *classA = [[ClassA alloc] init];
 unsigned int propertyCount;
 objc_property_t *result = class_copyPropertyList(object_getClass(classA), &propertyCount);
 for (unsigned int i = 0; i < propertyCount; i++) {
     objc_property_t objc_property_name = result[i];
     NSLog(@"%@",[NSString stringWithFormat:@"%s", property_getName(objc_property_name)]);
 }
 free(result);
 NSLog(@"--------------END----------------");
 NSLog(@"--- class_copyIvarList ↓↓↓---");
 Ivar *iv = class_copyIvarList(object_getClass(classA), &propertyCount);
 for (unsigned int i = 0; i < propertyCount; i++) {
     Ivar ivar = iv[i];
     NSLog(@"%@",[NSString stringWithFormat:@"%s", ivar_getName(ivar)]);
 }
 free(iv);
 NSLog(@"--------------END----------------");
}
```

以上[demo点击这里下载](https://github.com/sunyazhou13/IvarAndPropertyDemo)

---

参考 [objc 源码](https://github.com/sunyazhou13/objc-runtime) 可以进一步确认实现细节。

以下代码位于`objc-runtime-new.mm`中

```
/***********************************************************************
* class_copyPropertyList. Returns a heap block containing the
* properties declared in the class, or nil if the class
* declares no properties. Caller must free the block.
* Does not copy any superclass's properties.
* Locking: read-locks runtimeLock
**********************************************************************/
objc_property_t *
class_copyPropertyList(Class cls, unsigned int *outCount)
{
 if (!cls) {
     if (outCount) *outCount = 0;
     return nil;
 }

 mutex_locker_t lock(runtimeLock);

 checkIsKnownClass(cls);
 ASSERT(cls->isRealized());

 auto rw = cls->data();

 property_t **result = nil;
 unsigned int count = rw->properties.count();
 if (count > 0) {
     result = (property_t **)malloc((count + 1) * sizeof(property_t *));

     count = 0;
     for (auto& prop : rw->properties) {
         result[count++] = &prop;
 }
     result[count] = nil;
 }

 if (outCount) *outCount = count;
 return (objc_property_t *)result;
}
```

通过源码我们可以看到

```
auto rw = cls->data();
rw->properties; //通过rw直接拿到properties
```

通过rw直接拿到properties,然后便利拿出想要的 以`@property`关键字 声明变量名称.

`properties`详细内容 还请异步运行时源码看下这里篇幅限制就不啰嗦了.

---

```
/***********************************************************************
* class_copyIvarList
* fixme
* Locking: read-locks runtimeLock
**********************************************************************/
Ivar *
class_copyIvarList(Class cls, unsigned int *outCount)
{
 const ivar_list_t *ivars;
 Ivar *result = nil;
 unsigned int count = 0;

 if (!cls) {
     if (outCount) *outCount = 0;
     return nil;
 }

 mutex_locker_t lock(runtimeLock);

 ASSERT(cls->isRealized());

 if ((ivars = cls->data()->ro->ivars)  &&  ivars->count) {
     result = (Ivar *)malloc((ivars->count+1) * sizeof(Ivar));

     for (auto& ivar : *ivars) {
         if (!ivar.offset) continue;  // anonymous bitfield
         result[count++] = &ivar;
 }
     result[count] = nil;
 }

 if (outCount) *outCount = count;
 return result;
}
```
这里就一个关键点

```
ivars = cls->data()->ro->ivars
```

拿到ivars.

由于这两者拿到的成员不一样所以两个API就会有区别.

#### 1.11.15 `class_rw_t` 和 `class_ro_t` 的区别

先说结论:

- 两个结构体都存放着当前类的属性、实例变量、方法、协议等.
- `class_ro_t`存放的是编译期间就确定的.
- 而`class_rw_t`是在runtime时才确定，它会先将`class_ro_t`的内容拷贝过去，然后再将当前类的分类的这些属性、方法等拷贝到其中。所以可以说`class_rw_t`是`class_ro_t`的超集，当然实际访问类的方法、属性等也都是访问的`class_rw_t`中的内容.

---

#### 1.11.16 `class_rw_t` 与 `class_ro_t` 的来源

首先我们需要了解它俩的由来,在`objc_class`我们知道有一个成员变量叫`isa`,我们这里要介绍的是`objc_class`的另一成员变量`bits`.

`objc_class`的结构如下:

[![objc_class的结构](https://upload-images.jianshu.io/upload_images/13277235-aa0b11f641fa554d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/objc_class_struct.png)

`bits` 用来存储类的属性，方法，协议等信息。它是一个`class_data_bits_t`类型

`class_data_bits_t` 如下:

```
struct class_data_bits_t {
     uintptr_t bits;
     // method here
}
```

这个结构体只有一个`64bit`的成员变量`bits`，先来看看这`64bit`分别存放的什么信息：

[![](https://upload-images.jianshu.io/upload_images/13277235-0645dd043991b95d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/objc_class_bits.png)

- `is_swift` : 第一个bit，判断类是否是Swift类
- `has_default_rr` ：第二个bit，判断当前类或者父类含有默认的`retain/release/autorelease/retainCount/_tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference` 方法
- `require_raw_isa` ：第三个bit， 判断当前类的实例是否需要`raw_isa`
- `data` : 第4-48位，存放一个指向class_rw_t结构体的指针，该结构体包含了该类的属性，方法，协议等信息。至于为何只用44bit来存放地址

#### 1.11.17 `class_rw_t` 和`class_ro_t`

先来看看两个结构体的内部成员变量

```
struct class_rw_t {
     uint32_t flags;
     uint32_t version;

     const class_ro_t *ro;

     method_array_t methods;
     property_array_t properties;
     protocol_array_t protocols;

     Class firstSubclass;
     Class nextSiblingClass;
};
```

```
struct class_ro_t {
     uint32_t flags;
     uint32_t instanceStart;
     uint32_t instanceSize;
     uint32_t reserved;

     const uint8_t * ivarLayout;

     const char * name;
     method_list_t * baseMethodList;
     protocol_list_t * baseProtocols;
     const ivar_list_t * ivars;

     const uint8_t * weakIvarLayout;
     property_list_t *baseProperties;
};
```

`class_rw_t`结构体内有一个指向`class_ro_t`结构体的指针.

每个类都对应有一个`class_ro_t`结构体和一个`class_rw_t`结构体。在编译期间，`class_ro_t`结构体就已经确定，`objc_class`中的`bits`的`data`部分存放着该结构体的地址。在`runtime`运行之后，具体说来是在运行`runtime`的`realizeClass` 方法时，会生成`class_rw_t`结构体，该结构体包含了`class_ro_t`，并且更新`data`部分，换成`class_rw_t`结构体的地址。

用两张图来说明这个过程：

类的`realizeClass`运行之前：
[![](https://upload-images.jianshu.io/upload_images/13277235-5540d513f9d517e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/before_bits.png)

类的`realizeClass`运行之后：

[![](https://upload-images.jianshu.io/upload_images/13277235-a5e0786aaace5131.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/after_bits.png)

细看两个结构体的成员变量会发现很多相同的地方，他们都存放着当前类的属性、实例变量、方法、协议等等。区别在于：`class_ro_t`存放的是编译期间就确定的；而`class_rw_t`是在`runtime`时才确定，它会先将`class_ro_t`的内容拷贝过去，然后再将当前类的分类的这些属性、方法等拷贝到其中。所以可以说`class_rw_t`是`class_ro_t`的超集，当然实际访问类的方法、属性等也都是访问的`class_rw_t`中的内容

需要注意：实例变量（ivar）只存放在 `class_ro_t` 中，且运行期不可变；而属性、方法、协议在 `class_ro_t` 中保存的是编译期确定的基础部分（`baseProperties`、`baseMethods` 等），运行期则统一通过 `class_rw_t` 访问——其中合并了分类等动态添加的内容。

详细内容请 参考资料[Objective-C runtime - 属性与方法](http://vanney9.com/2017/06/05/objective-c-runtime-property-method/)

#### 1.11.18 Category 加载顺序与同名方法解析

结论:

1.  category 是 这样 `realizeClass` -> `methodizeClass()` -> `attachCategories()` 一步步被加载的.
2.  主类与分类的加载顺序是:**主类优先于分类加载,无关编译顺序**.
3.  分类间的加载顺序取决于编译的顺序:**编译在前则先加载,编译在后则后加载**.

---

#### 1.11.19 category如何被加载的

我在运行时的源码 `objc-runtime-new.mm`中找到如下:

```
static Class realizeClassWithoutSwift(Class cls, Class previously)
{
         ...
         // Attach categories  被加载
         methodizeClass(cls, previously);
         return cls;
}
```

`realizeClass` -> `methodizeClass()` -> `attachCategories()`

核心是在methodizeClass()函数中实现的.

```
static void methodizeClass(Class cls)
{
     runtimeLock.assertLocked();
     bool isMeta = cls->isMetaClass();
     auto rw = cls->data();
     auto ro = rw->ro;
     ...
     property_list_t *proplist = ro->baseProperties;
     if (proplist) {
         rw->properties.attachLists(&proplist, 1);
     }
     ...
     // Attach categories.
     category_list *cats =unattachedCategoriesForClass(cls, true /*realizing*/);
     attachCategories(cls, cats, false /*don't flush caches*/);
     ...
     if (cats) free(cats);

}
```

通过上述代码我们发现`ro->baseProperties;` , baseProperties 在前，category 在后,

```
property_list_t *proplist = ro->baseProperties;
if (proplist) {
  rw->properties.attachLists(&proplist, 1);
}
```
但决定顺序的是 rw->`properties.attachLists ()`这个方法.

```
/// category 被附加进去
void attachLists(List* const * addedLists, uint32_t addedCount) {
     if (addedCount == 0) return;
     if (hasArray()) {
         // many lists -> many lists
         uint32_t oldCount = array()->count;
         uint32_t newCount = oldCount +addedCount;
         setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
         array()->count = newCount;
 // 将旧内容移动偏移量 addedCount 然后将 addedLists copy 到起始位置
 /*
     struct array_t {
         uint32_t count;
         List* lists[0];
         };
 */
     memmove(array()->lists + addedCount,array()->lists,
             oldCount * sizeof(array()->lists[0]));
     memcpy(array()->lists, addedLists,
             addedCount * sizeof(array()->lists[0]));
 }
 else if (!list  &&  addedCount == 1) {
     // 0 lists -> 1 list
     list = addedLists[0];
 }
 else {
     // 1 list -> many lists
     List* oldList = list;
     uint32_t oldCount = oldList ? 1 : 0;
     uint32_t newCount = oldCount + addedCount;
     setArray((array_t*)malloc(array_t::byteSize(newCount)));
     array()->count = newCount;
     if (oldList) array()->lists[addedCount] = oldList;
     memcpy(array()->lists, addedLists,
     addedCount * sizeof(array()->lists[0]));
 }
}
```

所以 category 的属性总是在前面的，baseClass的属性被往后偏移了。

#### 1.11.20 两个category的load方法的加载顺序

```
A class’s +load method is called after all of its superclasses’ +load methods.
一个类的+load方法在其父类的+load方法后调用

A category +load method is called after the class’s own +load method.
一个Category的+load方法在被其扩展的类的自有+load方法后调用
```

结论: 主类与分类的加载顺序是:**主类优先于分类加载,无关编译顺序**.

#### 1.11.21 两个category的同名方法的加载顺序

应用程序 image 镜像加载到内存中时， `Category` 解析的过程，注意下面的 `while(i--)` 循环 这里倒序将 `category` 中的协议 方法 属性添加到了`rw = cls->data()`中的 `methods/properties/protocols`中。

```
static void
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
     if (!cats) return;
     if (PrintReplacedMethods) printReplacements(cls, cats);

     bool isMeta = cls->isMetaClass();

     // fixme rearrange to remove these intermediate allocations
     method_list_t **mlists = (method_list_t **)
         malloc(cats->count * sizeof(*mlists));
     property_list_t **proplists = (property_list_t **)
         malloc(cats->count * sizeof(*proplists));
     protocol_list_t **protolists = (protocol_list_t **)
         malloc(cats->count * sizeof(*protolists));

 // Count backwards through cats to get newest categories first
 int mcount = 0;
 int propcount = 0;
 int protocount = 0;
 int i = cats->count;
 bool fromBundle = NO;
 while (i--) {
     auto& entry = cats->list[i];

     method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
     if (mlist) {
         mlists[mcount++] = mlist;
         fromBundle |= entry.hi->isBundle();
     }

     property_list_t *proplist =
         entry.cat->propertiesForMeta(isMeta, entry.hi);
     if (proplist) {
         proplists[propcount++] = proplist;
     }

     protocol_list_t *protolist = entry.cat->protocols;
     if (protolist) {
         protolists[protocount++] = protolist;
     }
 }
 auto rw = cls->data();

 // 注意下面的代码，上面采用倒序遍历方式，所以后编译的 category 会先add到数组的前部
 prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
 rw->methods.attachLists(mlists, mcount);
 free(mlists);
 if (flush_caches  &&  mcount > 0) flushCaches(cls);

 rw->properties.attachLists(proplists, propcount);
 free(proplists);

 rw->protocols.attachLists(protolists, protocount);
 free(protolists);
}
```

所以结论是:分类间的加载顺序取决于编译的顺序:编译在前则先加载,编译在后则后加载

这个问题网上有很多例子 就不多在这举例了.

#### 1.11.22 Category 与 Extension 对比

#### 1.11.23 `category`

- 运行时添加分类属性/协议/方法
- 分类添加的方法会“覆盖”原类方法，因为方法查找的话是从头至尾，一旦查找到了就停止了
- 同名分类方法谁生效取决于编译顺序，image 读取的信息是倒序的，所以编译越靠后的越先读入
- 名字相同的分类会引起编译报错；

#### 1.11.24 `extension`

- 编译时决议
- 只以声明的形式存在，多数情况下就存在于 .m 文件中；
- 不能为系统类添加扩展

可以给类添加成员变量，但是是私有的 可以給类添加方法，但是是私有的 添加的属性和方法是类的一部分，在编译期就决定的。在编译器和头文件的@interface和实现文件里的@implement一起形成了一个完整的类。 伴随着类的产生而产生，也随着类的消失而消失

> **必须有类的源码才可以给类添加extension**!!!

#### 1.11.25 `category` & `extension`区别

- Category的小括号中有名字,而Extension没有;
- Category只能扩充方法,不能扩充成员变量和属性;
- 如果在 Category 中声明了一个属性，编译器只会生成该属性 setter / getter 方法的声明，而不会自动合成其实现，也不会生成对应的实例变量。所以对于系统一些类，如nsstring，就无法添加类扩展 不能给NSObject添加Extension，因为在extension中添加的方法或属性必须在源类的文件的.m文件中实现才可以，即：你必须有一个类的源码才能添加一个类的`extension`

#### 1.11.26 NSObject 添加 Extension 的限制

不能 因为没有NSObject的.m源码文件.

> 如果能的话那应该不叫Extension.或者我们自己通过运行时的api自己造一套ExtensionDIY.结果就是你用的根本不能称为`Extension`,而是api调用而已.

#### 1.11.27 消息转发机制，消息转发机制和其他语言的消息机制优劣对比

> 前言: 了解消息转发之前我们有必要了解一些Objectivce-C中的消息传递机制

#### 1.11.28 消息传递机制

在Objectivce-C中,我们通过`实例变量(对象)`或者`类方法名`调用一个方法,那么我们实际上是在发送一条消息

```
id returnValue = [someObject messageName:parameter];  //实例调用方式
id returnValue = [ClassA messageName:parameter];  //类调用方式
```

上述`someObject`和`ClassA`是接收者(receiver)，`messageName:`是选择器(`selector`),选择器和参数合起来称为消息(`message`)。编译器看到此消息后，将其转换为一条标准的c语言函数调用，所调用的函数乃是消息传递机制中的核心函数：`objc_msgSend()`。

```
id objc_msgSend(id self, SEL _cmd, ...); // 返回类型实际取决于被调方法，使用时按方法签名做相应转换
```
第一个参数代表接收者，第二个参数代表选择子，后续参数就是消息中的那些参数。编译器会把上面例子中的消息转换为如下函数调用：

```
id returnValue = objc_msgSend(someObject, @selector(messageName:),parameter);
id returnValue = objc_msgSend(ClassA, @selector(messageName:),parameter);
```

`objc_msgSend()`函数会依据接收者与选择器的类型来调用适当的方法.为来完成此操作，该方法需要在接收者所属的类中搜寻其“方法列表”(也就是上文我们说的`class_ro_t`中的method_list)。找到则跳到现实代码，否则，就沿着继承体系继续向上查找，如果还没有则执行消息转发操作。对于其他的“边界情况”，则需要交由Objective-c运行环境的另一些函数来处理：

```
objc_msgSend_stret  //待发送的消息返回结构体时
objc_msgSend_fpret  //消息返回的是浮点型
objc_msgSendSuper   //如果要给超类发送消息
```

#### 1.11.29 消息转发机制

结合上边的消息传递机制,在Objective-C中如果给一个对象发送一条它无法处理的消息，就会进入下图描述的消息转发(Message Forwarding)流程

[![](https://upload-images.jianshu.io/upload_images/13277235-1d51c512eecb61a5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/methodforward.jpg)

在objc中消息转发需要经历3个阶段 `resolveInstanceMethod` -> `forwardingTargetForSelector` -> `forwardInvocation` ->`消息未能处理`。

- 第一阶段:**动态方法解析(Dynamic Method Resolution)**也就是在所属的类中先征询接收者,看其是否能动态加方法，来处理当前这个**未知选择器**
- 第二阶段:**替换消息接收者快速转发**
- 第三阶段:**完全消息转发机制**

#### 1.11.30 第一阶段:**动态方法解析(Dynamic Method Resolution)**

对象在收到无法解读的消息后，首先将调用其所属类的下列类方法:

```
+ (BOOL)resolveClassMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
+ (BOOL)resolveInstanceMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```

> 这俩方法在NSObject.h中

返回一个`Boolean`类型，表示这个类是否能新增一个实例方法以处理选择器.

在 消息转发过程中,我们可以使用`resolveInstanceMethod:`动态的将一个方法添加到一个类中.

例下面示例代码:

```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
 if (aSEL == @selector(resolveThisMethodDynamically)) {
 class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
 return YES;
 }
 return [super resolveInstanceMethod:aSEL];
}
@end
```

这里我们用到一个运行时函数`class_addMethod()`.

```
{
 if (!cls) return NO;

 mutex_locker_t lock(runtimeLock);
 return ! addMethod(cls, name, imp, types ?: "", NO);
}
```

- `class_addMethod()`最后一个参数叫做`types`，是一个描述方法的参数类型的字符串.
- `v`代表`void`
- `@`代表对象或者说`id类型`
- `:`(这个冒号)代表方法选择器SEL

具体代表什么不是我们瞎写的,得按照苹果的这个标准 [Objective-C Runtime Programming Guide->Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

上面的`dynamicMethodIMP`，返回值是`void`，两个入参分别是`id`和`SEL`，所以描述这个方法的参数类型的字符串就是`v@:`

这个阶段的意义是为一个类动态提供方法实现,严格来说，还没进入消息转发流程。

`resolveInstanceMethod:` 控制这下面两个方法是否会被调用

- `respondsToSelector:`

- `instancesRespondToSelector:`

> 也就是说，如果`resolveInstanceMethod:`返回了`YES`，那么`respondsToSelector:`和`instancesRespondToSelector:`都会返回`YES`.

#### 1.11.31 第二阶段：替换消息接收者(快速转发)

如果第一阶段中`resolveInstanceMethod:`返回NO,就会调用`forwardingTargetForSelector:`询问是否把消息转发给另一个对象.消息的接收者就改变了。

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
 return someOtherObject;
}
```

#### 1.11.32 第三阶段：完全消息转发机制

如果第二阶段的`forwardingTargetForSelector:`返回了`nil`，这就进入了所谓完全消息转发的机制。

首先调用`methodSignatureForSelector:`为要转发的消息返回正确的签名：

```
- (void)forwardInvocation:(NSInvocation *)anInvocation {
     NSLog(@"forwardInvocation");
     SomeOtherObject *someOtherObject = [SomeOtherObject new];
     if ([someOtherObject respondsToSelector:[anInvocation selector]]) {
         [anInvocation invokeWithTarget:someOtherObject];
     } else {
         [super forwardInvocation:anInvocation];
     }
}
```

上面代码是将消息转发给其他对象，其实这与第二阶段中示例代码做的事情是一样的。区别就在于这个阶段会有一个`NSInvocation`对象。`NSInvocation`是一个用来存储和转发消息的对象。它包含了一个Objective-C消息的所有元素：一个target，一个selector，参数和返回值。每个元素都可以被直接设置。

> `NSInvocation`可以简单理解为一个对象把我们用到 selector方法和对象都存储了一下,然后哪个是指向我们需要调用的指针对象.

所以不同与第二阶段，在这个阶段你可以：

- 把消息存储，在你觉得合适的时机转发出去，或者不处理这个消息。
- 修改消息的target，selector，参数等
- 多次转发这个消息，转发给多个对象

显然在这个阶段，你可以对一个OC消息做更多的事情

---

#### 1.11.33 消息转发机制和其他语言的消息机制优劣对比

这个目前没有深入其它编程语言的运行时层面,比如C的底层或者C++的底层或者Java的底层消息传递

#### 1.11.34 在方法调用的时候，方法查询-> 动态解析-> 消息转发 之前做了什么

Objective-C 实例对象执行方法步骤

1.  获取 receiver 对应的类 Class
2.  在 Class 缓存列表中(就是`objc_class`里的`cache_t`到`class_ro_t`的方法list)根据选择子`selector`查找`IMP`
3.  若缓存中没有找到，则在方法列表中继续查找.
4.  若方法列表没有，则从父类查找，重复以上步骤.
5.  若最终没有找到，则进行消息转发操作.

- 方法查询之前 要知道 receiver和 selector.主要是要明确我们是哪个实例调用了哪个方法.

- 动态解析解析之前要 在所属的类中先征询接收者,看其是否能动态加方法，来处理当前这个未知选择器.
- 消息转发 之前 要询问是否把消息转发给另一个对象.

> 如果更深入的而理解 那应该是 objc_msgSend() 为啥是汇编实现的,上面的那些方法 调用之前 汇编的哪些指令被执行

这里找到两篇文章可以参考一下
[深入了解Objective-C消息发送与转发过程](https://chipengliu.github.io/2019/06/02/objc-msgSend-forward/)
[汇编语言编写的，其中具体过程细节](https://chipengliu.github.io/2019/04/07/objc-msg-armd64/)

#### 1.11.35 `IMP`、`SEL`、`Method`的区别和使用场景

- `IMP` : 是方法的具体实现(指针)

- `SEL` :方法名称

- `Method`:是objc_method类型指针，它是一个结构体 ,如下:

```
 struct objc_method {
     SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
     char * _Nullable method_types                            OBJC2_UNAVAILABLE;
     IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
     }
```

    使用场景
    
    - 例如 Button添加Target和Selector的时候.或者 实现类的`swizzle`的时候会用到，通过`class_getInstanceMethod(class, SEL)`来获取类的方法`Method`，其中用到了SEL作为方法名
    
    - 例如 给类动态添加方法，此时我们需要调用class_addMethod(Class, SEL, IMP, types)，该方法需要我们传递一个方法的实现函数IMP，例如:
    
    ``` objc
    static void funcName(id receiver, SEL cmd, 方法参数...) {
     // 方法具体的实现
    }
```

> SEL相当于 方法的类型 关键字.

#### 1.11.36 `load` 与 `initialize` 的区别

在Objective-C的类被加载和初始化的时候, 类 是 可以收到 方法回调的.

```
- (void)load;
- (void)initialize;
```

#### 1.11.37 `+load`

`+ load`方法是在这个文件(就是你复写的子类化的class)被程序装载时调用,只要是在Xcode `Compile Sources`中出现的文件总是会被装载，这与这个类是否被用到无关，因此+load方法总是在`main()`函数之前调用.

调用时机比较早，运行环境有不确定因素。具体说来，在iOS上通常就是App启动时进行加载，但当load调用的时候，并不能保证所有类都加载完成且可用，必要时还要自己负责做auto release处理。

> 补充上面一点，对于有依赖关系的两个库中，被依赖的类的+load会优先调用。但在一个库之内，父、子类、类别之间调用有顺序，不同类之间调用顺序是不确定的。

- 关于继承：对于一个类而言，没有+load方法实现就不会调用，不会考虑对NSObject的继承，就是不会沿用父类的+load。
- 父类和本类的调用：父类的方法优先于子类的方法。一个类的+load方法不用写明`[super load]`，父类就会收到调用。
- 本类和Category的调用：本类的方法优先于类别(Category)中的方法。Category的+load也会收到调用，但顺序上在本类的+load调用之后。
- 不会直接触发initialize的调用。

#### 1.11.38 `+initialize`

`+initialize`方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用，并且只会调用一次。`initialize`方法实际上是一种惰性(lazy load)调用，也就是说如果一个类一直没被用到，那它的initialize方法也不会被调用，这一点有利于节约资源.

runtime 使用了发送消息 `objc_msgSend` 的方式对 `+initialize` 方法进行调用。也就是说 `+initialize` 方法的调用与普通方法的调用是一样的，走的都是`发送消息的流程`。换言之，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 `+initialize`方法，那么就会对这个类中的实现造成覆盖(override)。

- initialize的自然调用是在第一次主动使用当前类的时候。
- 在initialize方法收到调用时，运行环境基本健全。
- 关于继承：和load不同，即使子类不实现initialize方法，会把父类的实现继承过来调用一遍，就是会沿用父类的+initialize。（沿用父类的方法中，self还是指子类）
- 父类和本类的调用：子类的+initialize将要调用时会激发父类调用的+initialize方法，所以也不需要在子类写明[super initialize]。(本着除主动调用外，只会调用一次的原则，如果父类的+initialize方法调用过了，则不会再调用)
- 本类和Category的调用：Category中的+initialize方法会覆盖本类的方法，只执行一个Category的+initialize方法。

`+load` 和 `+initialize` 对比如下：

|   | + load | + initialize |
| --- | :--: | :--: |
| 调用方式 | 直接使用函数内存地址 | objc_msgSend()方式 |
| 调用时机 | 被程序装载时调用main()函数之前,就是被添加到runtime时 | 在本类或它的子类收到第一条消息之前被调用 |
| 是否被系统单次调用(除主动调用外) | 是 | 是 |
| 运行时环境是否稳定 | 不确定 | 稳定 |
| 线程是否安全 | 默认是安全的(已加锁) | 安全(已加锁 ) |
| 特性 | 由于非`objc_msgSend()`方式调用就使得 +load 方法拥有了一个非常有趣的特性，那就是子类、父类和分类中的 +load 方法的实现是被区别对待的。也就是说如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的。同理，当一个类和它的分类都实现了 +load 方法时，两个方法都会被调用 | +initialize 方法的调用与普通方法的调用是一样的，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 +initialize 方法，那么就会对这个类中的实现造成覆盖 |

参考[类方法load和initialize的区别](https://cloud.tencent.com/developer/article/1355957)

#### 1.11.39 继承关系中的调用差异

super的方法会成功调用，但是这是多余的，因为runtime会自动对父类的+load方法进行调用，而+initialize则会随子类自动激发父类的方法（如Apple文档中所言）不需要显式调用。另一方面，如果父类中的方法用到的self（像示例中的方法），其指代的依然是类自身，而不是父类

#### 1.11.40 说说消息转发机制的优劣

优点:

- 利用消息转发机制可以无代码侵入的实现多重代理，让不同对象可以同时代理同个回调，然后在各自负责的区域进行相应的处理，降低了代码的耦合程度。

- 使用 @synthesize 可以为 @property 自动生成 getter 和 setter 方法（现 Xcode 版本中，会自动生成），而 @dynamic 则是告诉编译器，不用生成 getter 和 setter 方法。当使用 @dynamic 时，我们可以使用消息转发机制，来动态添加 getter 和 setter 方法。当然你也用其他的方法来实现。

缺点:

- Objective-C本身不支持多继承，这是因为消息机制名称查找发生在运行时而非编译时，很难解决多个基类可能导致的二义性问题，但是可以通过消息转发机制在内部创建多个功能的对象，把不能实现的功能给转发到其他对象上去，这样就做出来一种多继承的假象。转发和继承相似，可用于为OC编程添加一些多继承的效果，一个对象把消息转发出去，就好像他把另一个对象中方法接过来或者“继承”一样。消息转发弥补了objc不支持多继承的性质，也避免了因为多继承导致单个类变得臃肿复杂。


### 1.12 weak的实现原理？SideTable的结构是什么样的

先说结论:

- weak 表本质上是一个哈希表：`Key` 是所指对象的地址，`Value` 是指向该对象的所有 weak 指针地址组成的数组。其实现是将被弱引用的对象统一登记到 `SideTable` 中 `weak_table_t` 类型的 `weak_table` 里，并通过 `objc_initWeak()` → `storeWeak()` 的调用链，借助新旧两张 `SideTable` 完成弱引用的注册与迁移。
- `SideTable`是一个结构体，内部主要有引用计数表和弱引用表两个成员，内存存储的其实都是对象的地址和引用计数和weak变量的地址，而不是对象本身的数据,它的结构如下

|

```
struct SideTable {
     spinlock_t slock;
     RefcountMap refcnts;
     weak_table_t weak_table;
     SideTable() {
         memset(&weak_table, 0, sizeof(weak_table));
 }
     ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
 }
     void lock() { slock.lock(); }
     void unlock() { slock.unlock(); }
     void forceReset() { slock.forceReset(); }
     // Address-ordered lock discipline for a pair of side tables.
     template<HaveOld, HaveNew>
     static void lockTwo(SideTable *lock1,SideTable *lock2);
     template<HaveOld, HaveNew>
     static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

#### 1.12.1 weak实现原理

实现原理概括分为3个时机

- 1.初始化
- 2.添加引用
- 3.释放

#### 1.12.2 初始化时候

`runtime`会调用`objc_initWeak`函数，初始化一个新的`weak`指针指向对象的地址.

我们引入一段测试代码

```
NSObject *obj = [[NSObject alloc] init];
id __weak obj1 = obj;
```

当我们初始化一个weak变量时，`runtime`会调用`NSObject.mm`中的`objc_initWeak()`函数。这个函数在Clang中的声明如下：

```
id objc_initWeak(id *location, id newObj) {
     if (!newObj) { // 查看对象实例是否有效 无效对象直接导致指针释放
         *location = nil;
         return nil;
 }
     // 这里传递了三个 bool 数值 old, new, crash.使用 template 进行常量参数传递是为了优化性能
     return storeWeak<DontHaveOld, DoHaveNew, DontCrashIfDeallocating>
         (location, (objc_object*)newObj);
}
```

可以看出，这个函数仅仅是一个深层函数的调用入口，而一般的入口函数中，都会做一些简单的判断（例如 `objc_msgSend` 中的缓存判断），这里判断了其指针指向的类对象是否有效，无效直接释放，不再往深层调用函数。否则，object将被注册为一个指向value的`__weak`对象。而这事应该是`objc_storeWeak`函数干的.

> 注意： `objc_initWeak`函数有一个前提条件：就是object必须是一个没有被注册为`__weak`对象的有效指针。而value则可以是null，或者指向一个有效的对象.

#### 1.12.3 添加引用时

`objc_initWeak`函数会调用 `objc_storeWeak()`函数,`objc_storeWeak()`则会调用`storeWeak()`函数， `storeWeak()`的作用是更新指针指向，创建对应的弱引用表

模板

```
// HaveOld:  true - 变量有值 ,false - 需要被及时清理，当前值可能为 nil
// HaveNew:  true - 需要被分配的新值，当前值可能为nil, false - 不需要分配新值
// CrashIfDeallocating: true - 说明 newObj 已经释放或者 newObj 不支持弱引用，该过程需要暂停,false - 用 nil 替代存储
template <HaveOld haveOld, HaveNew haveNew,CrashIfDeallocating crashIfDeallocating>
```

weak实现函数 **该过程用来更新弱引用指针的指向**.

```
static id
storeWeak(id *location, objc_object *newObj)
{
    ASSERT(haveOld  ||  haveNew);
    if (!haveNew) ASSERT(newObj == nil);
    // 初始化 previouslyInitializedClass 指针.
    Class previouslyInitializedClass = nil;
    id oldObj;
    // 声明两个 SideTable,① 新旧散列创建
    SideTable *oldTable;
    SideTable *newTable;
    //获得新值和旧值所在 SideTable 的位置(以对象地址作为唯一标识),通过地址建立索引,下面的操作会改变旧值.
    if (haveOld) {
        oldObj = *location;// 更改指针，获得以 oldObj 为索引所存储的值地址
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];// 更改新值指针，获得以 newObj 为索引所存储的值地址
    } else {
        newTable = nil;
    }
    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
	// 避免线程冲突重处理,location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已经处理过 oldObj 可是又被其他线程所修改
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }
    // 防止弱引用间死锁,并且通过 +initialize 初始化构造器保证所有弱引用的 isa 非空指向
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();// 获得新对象的 isa 指针
        // 判断 isa 非空且已经初始化
        if (cls != previouslyInitializedClass  &&
            !((objc_class *)cls)->isInitialized())
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);/ 解锁
            class_initialize(cls, (id)newObj); //如果该类已经完成执行 +initialize 方法是最理想情况,如果该类 +initialize 在线程中,例如 +initialize 正在调用 storeWeak 方法,需要手动对其增加保护策略，并设置 previouslyInitializedClass 指针进行标记
            previouslyInitializedClass = cls;
            goto retry; //重试
        }
    }
    // ② 清除旧值
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }
	 // ③ 分配新值
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location,
                                  crashIfDeallocating);
        //如果弱引用被释放 weak_register_no_lock 方法返回 nil,在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
	        //弱引用位初始化操作,引用计数那张散列表的weak引用对象的引用计数中标识为weak引用
            newObj->setWeaklyReferenced_nolock();
        }
        //之前不要设置 location 对象，这里需要更改指针指向
        *location = (id)newObj;
    }
    else {
        // 没有新值，则无需更改
    }

    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
    
    return (id)newObj;
}
```

#### 1.12.4 SideTable

SideTable就是一个结构体，内部主要有引用计数表和弱引用表两个成员，内存存储的其实都是对象的地址和引用计数和weak变量的地址，而不是对象本身的数据. > 主要用于管理对象的引用计数和 weak 表.

我们来看图

[![](https://upload-images.jianshu.io/upload_images/13277235-8dee1bc3b238d3f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/SideTableStructure.png)

> 操作系统维护64个SideTable，通过对象的地址位置hash之后模64(就是%64求余数)找到指定的SideTable 每个SideTable维护了一个RefcountMap的引用计数表，key就是对象地址，value就是此对象的引用计数

```
struct SideTable {
    spinlock_t slock; //保证原子操作的自旋锁
    RefcountMap refcnts; //引用计数的 hash 表
    weak_table_t weak_table; //weak 引用全局 hash 表
    ...
};
```

- slock 防止竞争的自旋锁
- refcnts 协助对象的 isa 指针的`extra_rc`共同引用计数的变量

#### 1.12.5 weak表

弱引用hash表,`weak_table_t`类型的结构体,存储某个实例对象相关的所有弱引用信息. 定义如下:

```
struct weak_table_t {
    weak_entry_t *weak_entries; // 保存了所有指向指定对象的 weak 指针
    size_t    num_entries;		 // 当前已存储的 weak_entry_t 条目数
    uintptr_t mask;     			// 哈希表容量掩码（容量减一），用于按位与取模
    uintptr_t max_hash_displacement;     // 哈希冲突时的最大探测偏移值
};
```

这是一个全局弱引用hash表。使用不定类型对象的地址作为`key`，用`weak_entry_t`类型结构体对象作为`value`,其中的`weak_entries` 成员,即为弱引用表入口.

其中`weak_entry_t`是存储在弱引用表中的一个内部结构体，它负责维护和存储指向一个对象的所有弱引用hash表。其定义如下：

```
typedef DisguisedPtr<objc_object *> weak_referrer_t;
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
    ...
};
```

其中`referent`是`DisguisedPtr<objc_object>`类型,它是**对对象指针的伪装封装**,将指针值做位运算变形后再存储,以避免被内存检测工具误判为对该对象的强引用而造成泄漏误报.

这里有一个很重要的标志字段`out_of_line_ness`,它占据`inline_referrers[1]`的低2位:当它为0时,使用内联数组`inline_referrers`直接存放weak指针;不为0时,则改用外置(out-of-line)的哈希表`referrers`存放.

其中`weak_referrer_t` 是 `DisguisedPtr<objc_object *>` 的别名(typedef),即对weak指针(二级指针)的伪装封装;这些referrer以数组形式存放,并通过哈希定位,构成一张弱引用散列表。

当采用外置哈希表（out-of-line）存储时，`weak_entry_t` 中各成员的作用如下：

- `out_of_line_ness`：标志位。为 0 时使用内联数组 `inline_referrers`，否则使用外置哈希表 `referrers`。
- `num_refs`：记录哈希表中有效弱引用的数量。
- `mask`：哈希表容量掩码（容量减一），用于按位与取模定位槽位。
- `max_hash_displacement`：哈希冲突时允许的最大线性探测偏移。

> 通常情况下 `out_of_line_ness` 为 0，此时一个 `weak_entry_t` 使用容量为 `WEAK_INLINE_COUNT`（默认 4）的内联数组 `inline_referrers` 直接存放指向该对象的 weak 指针；只有当 weak 指针数量超过该上限时，才切换为外置的哈希表 `referrers`。

以上是weak表的实现原理.

#### 1.12.6 释放

释放时，调用`clearDeallocating`函数。`clearDeallocating`函数首先根据对象地址获取所有`weak`指针地址的数组，然后遍历这个数组把其中的数据设为`nil`，最后把这个`entry`从`weak`表中删除，最后清理对象的记录.

#### 1.12.7 弱引用对象释放流程

- 1.调用`objc_release`
- 2.因为对象的引用计数为0，所以执行`dealloc`
- 3.在dealloc中，调用了`_objc_rootDealloc`函数
- 4.在`_objc_rootDealloc`中，调用了`object_dispose`函数
- 5.调用`objc_destructInstance`
- 6.最后调用`objc_clear_deallocating`

重点看对象被释放时调用的`objc_clear_deallocating`函数。该函数实现如下:

```
void objc_clear_deallocating(id obj)
{
    ASSERT(obj);
    if (obj->isTaggedPointer()) return;
    obj->clearDeallocating();
}
```

调用了`clearDeallocating()`,点击源码进去追踪发现,它最终是使用了迭代器来取`weak`表的`value`,然后调用`weak_clear_no_lock()`查找对应`value`,将该`weak`指针置空.

`weak_clear_no_lock()`函数的实现如下:

```
void weak_clear_no_lock(weak_table_t *weak_table, id referent_id)
{
    objc_object *referent = (objc_object *)referent_id;
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }
    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    }
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n",
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    weak_entry_remove(weak_table, entry);
}
```

`objc_clear_deallocating()`该函数的动作如下：

- 1.从weak表中获取废弃对象的地址为键值的记录
- 2.将包含在记录中的所有附有 weak修饰符变量的地址，赋值为nil
- 3.将weak表中该记录删除
- 4.从引用计数表中删除废弃对象的地址为键值的记录

[参考](https://www.jianshu.com/p/13c4fb1cedea)

### 1.13 关联对象的应用？系统如何实现关联对象的

#### 1.13.1 关联对象的应用？

一般应用在`category`(分类)中为 当前类 添加关联属性,因为不能直接添加成员变量，但是可以通过runtime的方式间接实现添加成员变量的效果。

当我们在`category`中声明如下代码:

```
@interface ClassA (Category)
@property (nonatomic, strong) NSString *property;
@end
```

实际上`@property`这个objc标准库的内建关键字帮我们实现了 setter和 getter,但是在category中并不能帮我们声明成员变量 `property` 我们需要通过runtime提供的两个C函数的api间接实现 动态添加 成员变量`property`.

- `objc_setAssociatedObject()`
- `objc_getAssociatedObject()`

```
#import "ClassA+Category.h"
#import <objc/runtime.h>

@implementation ClassA (Category)

- (NSString *) property {
    return objc_getAssociatedObject(self, _cmd);
    }

- (void)setProperty:(NSString *)categoryProperty {
    objc_setAssociatedObject(self, @selector(property), categoryProperty, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }

@end
```

看到上面的关联方法,我们来仔细研究一下下面经常使用的关联属性相关的API

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
```

1.  `objc_setAssociatedObject()`以键值对形式添加关联对象
2.  `objc_getAssociatedObject()`根据 key 获取关联对象
3.  `objc_removeAssociatedObjects()`移除所有关联对象

`objc_setAssociatedObject()`的调用栈

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
└── SetAssocHook.get()(object, key, value, policy)
    └── void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy)

```

上述调用栈中的`_object_set_associative_reference()`函数实际完成了设置关联对象的任务：

```
void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
     if (!object && !value) return;
    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value};
    association.acquireValue();
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
            if (refs_result.second) {
                object->setHasAssociatedObjects();
            }
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            ...
        }
    }
    association.releaseHeldValue();
}
```

省略的很多代码,上述代码中就是应用场景,上面调用的类`AssociationsManager`就是我们下面要讲的系统如何实现关联对象的原理.

#### 1.13.2 系统如何实现关联对象的(关联对象实现原理)

实现关联对象技术的核心对象 有如下这么几个:

1.  AssociationsManager
2.  AssociationsHashMap

3.  ObjectAssociationMap
4.  ObjcAssociation

> 其中Map同我们平时使用的字典类似。通过`key`-`value`的形式对应存值.

可以通过源码继续分析其内部实现。

#### 1.13.3 `objc_setAssociatedObject()`函数

runtime源码

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
{
    _object_set_associative_reference(object, key, value, policy);
}
```

> 源码调用过程有hook函数,有点长,这里我简化一下,直接调用核心的函数

下面看下`_object_set_associative_reference()`函数的代码实现

```
void _object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value}; //4. 我们用到的ObjcAssociation
    association.acquireValue();
    {
        AssociationsManager manager; //1. 我们用到的AssociationsManager
        AssociationsHashMap &associations(manager.get()); //2.我们上面列举的AssociationsHashMap
        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{}); //3.我们用到的ObjectAssociationMap
            if (refs_result.second) {
                object->setHasAssociatedObjects();
            }
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            auto refs_it = associations.find(disguised);
            if (refs_it != associations.end()) {
                auto &refs = refs_it->second;
                auto it = refs.find(key);
                if (it != refs.end()) {
                    association.swap(it->second);
                    refs.erase(it);
                    if (refs.size() == 0) {
                        associations.erase(refs_it);
                    }
                }
            }
        }
    }
    association.releaseHeldValue();
}
```

上述代码可以找到实现关联对象技术的核心对象，接下来分别说明几个核心对象的内部实现。

#### 1.13.4 AssociationsManager

```
typedef DenseMap<const void *, ObjcAssociation> ObjectAssociationMap;
typedef DenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap> AssociationsHashMap;
class AssociationsManager {
    using Storage = ExplicitInitDenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap>;
    static Storage _mapStorage;

public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }

    AssociationsHashMap &get() {
        return _mapStorage.get();
    }
    static void init() {
        _mapStorage.init();
    }
};
```

`AssociationsManager`内部有一个`get()`函数返回一个`AssociationsHashMap`对象

#### 1.13.5 AssociationsHashMap

`AssociationsHashMap` 是`DenseMap`的typedef(可以理解为别名) 只不过它被定义成符合某些`元组`的条件的`DenseMap`类型

实际上 `AssociationsHashMap` 用于保存从对象的 `DisguisedPtr` 到 `ObjectAssociationMap` 的映射，这一数据结构保存了当前对象对应的所有关联对象

```
typedef DenseMap<const void *, ObjcAssociation> ObjectAssociationMap;
typedef DenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap> AssociationsHashMap;

```

这里的`ObjectAssociationMap`是另一类型的typedef,里面存着`ObjcAssociation`类型的对象指针的key,value形式.

下面再看下 `ObjcAssociation` ,这是一个C++的类对象,最关键的`ObjcAssociation`包含了`policy`以及`value`.

```
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
public:
    ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
    ObjcAssociation() : _policy(0), _value(nil) {}
    ObjcAssociation(const ObjcAssociation &other) = default;
    ObjcAssociation &operator=(const ObjcAssociation &other) = default;
    ObjcAssociation(ObjcAssociation &&other) : ObjcAssociation() {
        swap(other);
    }
    inline void swap(ObjcAssociation &other) {
        std::swap(_policy, other._policy);
        std::swap(_value, other._value);
    }
    inline uintptr_t policy() const { return _policy; }
    inline id value() const { return _value; }
    ...
};
```

#### 1.13.6 关联对象在内存中以什么形式存储的？

示例代码

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *obj = [NSObject new];
        objc_setAssociatedObject(obj, @selector(hello), @"Hello", OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return 0;
}
```

这个调用函数`objc_setAssociatedObject(OBJC_ASSOCIATION_RETAIN_NONATOMIC, @"Hello")`在内存中是这样的存储结构

[![](https://upload-images.jianshu.io/upload_images/13277235-d5bf35501e7cf17e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/AssociationOrder.png)

#### 1.13.7 `objc_setAssociatedObject()`

我们回头来详细分解一下`objc_setAssociatedObject()`函数中的真实实现部分,`_object_set_associative_reference()`

这个函数需要传入`(id object, const void *key, id value, uintptr_t policy)`,这么几个参数,我们拿第3个`value`参数来分解.

我们分解为2步

1.  `value != nil` 设置或者更新关联对象的值
2.  `value == nil` 删除一个关联对象.

下面是具体是代码解释 **注意看代码注释!!!**

```
void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    // 判空
    if (!object && !value) return;

	// 判断本类对象是否允许关联其他对象.如果允许则进入代码块
	if (object->getIsa()->forbidsAssociatedObjects())
	    _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
	
	// 将被关联的对象封装成DisguisedPtr方便在后边hash表中的管理,它的作用就像是一个指针
	DisguisedPtr<objc_object> disguised{(objc_object *)object};
	// 将需要关联的对象,封装成ObjcAssociation,方便管理
	ObjcAssociation association{policy, value};
	
	// 处理policy为retain和copy的修饰情况,
	association.acquireValue();
	
	{
		// 获取关联对象管理者对象
	    AssociationsManager manager;
	    // 根据管理者对象获取对应关联表(HashMap)
	    AssociationsHashMap &associations(manager.get());
	
	    if (value) {
	    	// 如果这个disguised存在于ObjectAssociationMap()中,则替换,如果不存在则初始化后在插入
	    	// 这里说明一下,我们关联的对象关系存在于ObjectAssociationMap中,而
	    	//	ObjectAssociationMap有多个,所以,这一步是对ObjectAssociationMap的一个管理,下边才是对我们要关联的对象的操作
	        auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
	        // 如果这是此对象第一次被关联
	        if (refs_result.second) {
	           // 修改isa_t中的has_assoc字段,标记其被关联状态
	            object->setHasAssociatedObjects();
	        }
	
	        // 这里才是对我们要关联的对象操作
	        auto &refs = refs_result.first->second;
	        // 想map中插入key value对
	        auto result = refs.try_emplace(key, std::move(association));
	        // 这里没有看懂,为什么没有第二个就要交换一下..
	        if (!result.second) {
	            association.swap(result.first->second);
	        }
	    } else {
	    	// value为空, 并且在associations中有记录,则进行擦除操作
	        auto refs_it = associations.find(disguised);
	        if (refs_it != associations.end()) {
	            auto &refs = refs_it->second;
	            auto it = refs.find(key);
	            if (it != refs.end()) {
	                association.swap(it->second);
	                refs.erase(it);
	                if (refs.size() == 0) {
	                    associations.erase(refs_it);
	                }
	            }
	        }
	    }
	}
	
	// release the old value (outside of the lock).
	association.releaseHeldValue();
}
```

#### 1.13.8 `objc_setAssociatedObject()`函数的作用是什么?

```
inline void
objc_object::setHasAssociatedObjects()
{
    if (isTaggedPointer()) return;

 retry:
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    if (!newisa.nonpointer  ||  newisa.has_assoc) {
        ClearExclusive(&isa.bits);
        return;
    }
    newisa.has_assoc = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}
```

它会将`isa`结构体中的标记位`has_assoc`标记为`true`，也就是表示当前对象有关联对象，如下图`isa`中的各个标记位都是干什么的.

[![](https://upload-images.jianshu.io/upload_images/13277235-47b4c390532bf401.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/isa.jpg)

#### 1.13.9 `objc_getAssociatedObject()`

这个函数的调用栈如下

```
id objc_getAssociatedObject(id object, const void *key)
└── id _object_get_associative_reference(id object, const void *key);
```

通过上面我们介绍，理解这个函数相当简单了

```
id
_object_get_associative_reference(id object, const void *key)
{
    ObjcAssociation association{};
    {
        AssociationsManager manager; //1
        AssociationsHashMap &associations(manager.get()); //1
        AssociationsHashMap::iterator i = associations.find((objc_object *)object); //2
        if (i != associations.end()) {
            ObjectAssociationMap &refs = i->second;
            ObjectAssociationMap::iterator j = refs.find(key);
            if (j != refs.end()) {
                association = j->second;
                association.retainReturnedValue();
            }
        }
    }
    return association.autoreleaseReturnedValue();
}
```

1.  通过`AssociationsManager`拿到`AssociationsHashMap`哈希表
2.  通过哈希表寻找关联对象
3.  剩下的就是更新对象是否初次创建等标记 然后返回对象

#### 1.13.10 `objc_removeAssociatedObjects()`

调用栈如下:

```
void objc_removeAssociatedObjects(id object)
└── void _object_remove_assocations(id object)
```

代码具体实现

```
void objc_removeAssociatedObjects(id object)
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}
```

> check对象是否为nil 且 关联对象是否存在

然后调用实现跟上边的get差不多

```
void
_object_remove_assocations(id object)
{
    ObjectAssociationMap refs{};
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            refs.swap(i->second);
            associations.erase(i);
        }
    }
    // release everything (outside of the lock).
    for (auto &i: refs) {
        i.second.releaseHeldValue();
    }
}
```

通过`AssociationsManager` -> `AssociationsHashMap` -> object 是否存在,如果存在就**擦除**.- > releaseHeldValue()是否对象

#### 1.13.11 小结

关联对象的应用和系统如何实现关联对象的大概顺序如下:
`AssociationsManager`关联对象管理器->`AssociationsHashMap`哈希映射表->`ObjectAssociationMap`关联对象指针->`ObjcAssociation`关联对象

### 1.14 关联对象的如何进行内存管理的？关联对象如何实现weak属性?

#### 1.14.1 关联对象的如何进行内存管理的？

当我调用关联对象函数`objc_setAssociatedObject()`的时候会调用如下函数：

`_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)`,这里面有个方法

```
ObjcAssociation association{policy, value};
// retain the new value (if any) outside the lock.
association.acquireValue();
```

这里的 `policy`就是具体绝对内存使用retain还是其它相关的内存枚举.

```
enum {
    OBJC_ASSOCIATION_SETTER_ASSIGN      = 0,
    OBJC_ASSOCIATION_SETTER_RETAIN      = 1,
    OBJC_ASSOCIATION_SETTER_COPY        = 3,            // NOTE:  both bits are set, so we can simply test 1 bit in releaseValue below.
    OBJC_ASSOCIATION_GETTER_READ        = (0 << 8),
    OBJC_ASSOCIATION_GETTER_RETAIN      = (1 << 8),
    OBJC_ASSOCIATION_GETTER_AUTORELEASE = (2 << 8)
};
```

通过 acquireValue()函数判断使用那种内存关键字.

```
inline void acquireValue() {
    if (_value) {
        switch (_policy & 0xFF) {
        case OBJC_ASSOCIATION_SETTER_RETAIN:
            _value = objc_retain(_value);
            break;
        case OBJC_ASSOCIATION_SETTER_COPY:
            _value = ((id(*)(id, SEL))objc_msgSend)(_value, @selector(copy));
            break;
        }
    }
}
```

#### 1.14.2 关联对象如何实现weak属性？

首先说一下 这个问题问的非常有技术含量,完全考验iOS开发者对底层了解的程度.

在为NSObject对象绑定 associated object 时可以指定如下依赖关系：

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0, //弱引用
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, //强引用，非原子操作
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,  //先 copy，然后强引用
    OBJC_ASSOCIATION_RETAIN = 01401, //强引用，原子操作
    OBJC_ASSOCIATION_COPY = 01403 //先 copy，然后强引用，原子操作
};
```

根据上述的枚举我们发现一个很奇怪的问题,这里的枚举中并没有`OBJC_ASSOCIATION_WEAK`这样的选项.

基于上述的代码介绍我们知道`Objective-C`在底层使用`AssociationsManager`统一管理各个对象的 `associated objects`关联对象.然后通过`static key`(一般是一个固定值)去访问对应的`associated object`关联对象.然后在`dealloc`的时候调用`擦除函数`(`associations.erase()`)来解除对这些关联对象的引用:

```
dealloc
    object_dispose
        objc_destructInstance
            _object_remove_assocations  // 移除必要的associated objects
```

也就是说,在`NSObject`对象的内存空间里，并没有为 `associated objects`(关联对象) 分配任何变量.

我们知道weak变量和 assign变量的区别是:weak指向的对象销毁的时候,`Objective-C`会自动帮我们设置`nil`,而`assign`却不能.

这个逻辑是如何实现的呢？

`Runtime` 在底层维护一张 weak 表（即前文所讲 `SideTable` 中 `weak_table_t` 类型的 `weak_table`）。每当为一个 weak 指针赋上有效对象的地址时，就会把对象地址和该 weak 指针地址注册到 weak 表中，其中对象地址作为 key；当对象被废弃时，可根据对象地址快速找到指向它的所有 weak 指针，将它们置为 nil 并从 weak 表中移除。

所以,实现`weak`引用(而非`assign`引用)的前提是存在一个`__weak`指针指向到被引用对象的地址,只有这样,当对象被销毁时，指针才能被`runtime`找到然后被设置为`nil`；`NSObject`对象和其`associated object`关联对象的关系，并不存在指针这样的**中间媒介**，因此只存在`OBJC_ASSOCIATION_ASSIGN`选项，而不存在`OBJC_ASSOCIATION_WEAK`选项.

#### 1.14.3 关联对象 weak 属性的实现方案

可以通过曲线救国的方式声明一个`class`类 持有一个weak的成员变量,然后通过 实例化 我们自定义的class的实例,然后把这个实例作为关联对象即可.

声明封装weak对象的类

```
@interface WeakAssociatedObjectWrapper : NSObject
@property (nonatomic, weak) id object;
@end

@implementation WeakAssociatedObjectWrapper
@end
```

调用

```
@interface UIView (ViewController)
@property (nonatomic, weak) UIViewController *vc;
@end

@implementation UIView (ViewController)
- (void)setVc:(UIViewController *)vc {
    WeakAssociatedObjectWrapper *wrapper = [WeakAssociatedObjectWrapper new];
    wrapper.object = vc;
    objc_setAssociatedObject(self, @selector(vc), wrapper, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
- (UIViewController *)vc {
    WeakAssociatedObjectWrapper *wrapper = objc_getAssociatedObject(self, _cmd);
    return wrapper.object;
    }
    @end
```

> 看明白没有,曲线救国.代码引入自[Weak Associated Object](https://zhangbuhuai.com/post/weak-associated-object.html)

[关联对象参考](https://draveness.me/ao/)

### 1.15 Autoreleasepool的原理？所使用的的数据结构是什么？

在ARC下我们使用`@autoreleasepool{}` 关键字 把需要自动管理的代码块圈起来 ,这个过程就是在使用一个`AutoReleasePool`

```
@autoreleasepool {
	 <#statements#> //代码块
}
```

以上代码经编译器改写后，会在代码块首尾分别插入 push 与 pop，形如：

```
void *atautoreleasepoolobj = objc_autoreleasePoolPush();
// ... 代码块 ...
objc_autoreleasePoolPop(atautoreleasepoolobj);
```

既然有压栈，就一定有对应的出栈操作 `objc_autoreleasePoolPop()`。其中 push 返回的是本次自动释放池的哨兵对象（`POOL_BOUNDARY`）地址，pop 时再传回该地址，以确定释放到哪一层为止。

- `objc_autoreleasePoolPush()`
- `objc_autoreleasePoolPop()`

这俩函数都是对`AutoreleasePoolPage`的封装,自动释放机制的核心就是这个类

#### 1.15.1 `AutoreleasePoolPage`

`AutoreleasePoolPage`是个C++的类

[![](https://upload-images.jianshu.io/upload_images/13277235-a112c7bd28593aa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/autoreleasepoolpage.png)

- **AutoreleasePool**并没有单独的结构,而是由若干个`AutoreleasePoolPage`以`双向链表`的形式组合成的,根据上图可以看出,这个双向链表有`前驱parent`和`后继child`.
- **AutoreleasePool**是按`线程`一一对应的(thread 成员变量)
- **AutoreleasePoolPage** 是自动释放池存储对象的数据结构。每个 Page 的大小为 `PAGE_MAX_SIZE`（64 位真机上通常为 16KB，模拟器等为 4KB），其中开头若干字节用于存放自身的成员变量，其余空间用来存放调用了 `autorelease` 的对象地址；嵌套的自动释放池之间通过插入哨兵对象（`POOL_BOUNDARY`，其值为 nil）来标记边界
- 当一个page被占满以后会新建一个新的`AutoreleasePoolPage`对象,并插入哨兵标记. 具体代码如下:

```
class AutoreleasePoolPage {
//##   define EMPTY_POOL_PLACEHOLDER ((id*)1)
//##   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

- `magic` 检查校验完整性的变量
- `next` 指向新加入的autorelease对象
- `thread` page当前所在的线程，AutoreleasePool是按线程一一对应的（结构中的thread指针指向当前线程）
- `parent` 父节点 指向前一个page
- `child` 子节点 指向下一个page
- `depth` 链表的深度，节点个数
- `hiwat` high water mark 数据容纳的一个上限
- `EMPTY_POOL_PLACEHOLDER` 空池占位
- `POOL_BOUNDARY` 是一个边界对象 nil,之前的源代码变量名是 `POOL_SENTINEL`哨兵对象,用来区别每个page即每个 AutoreleasePoolPage 边界
- `PAGE_MAX_SIZE` 的取值与系统虚拟内存页大小对齐（经典值为 4KB，arm64 真机上为 16KB），以便按内存页对齐管理。
- `COUNT` 一个page里对象数

下面看下工作机制图

[![](https://upload-images.jianshu.io/upload_images/13277235-4d8656f7b69c5dba.gif?imageMogr2/auto-orient/strip)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/autoreleasepoolworkflow.gif)

根据上面的示意图我们大概明白, `AutoreleasePoolPage`是以栈的形式存在,并且内部对象通过进栈出栈来对应着`objc_autoreleasePoolPush`和`objc_autoreleasePoolPop`

如果嵌套AutoreleasePool 就是通过`哨兵对象`来标识,每次更新链表的next和`前驱``后继`来完成表的创建销毁.

[![](https://upload-images.jianshu.io/upload_images/13277235-82a0c2edb402845f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/autoreleasepoolpage1.png)

当我们对一个对象发送一条`autorelease`消息的时候实际上就是将这个对象加入到当前`AutoreleasePoolPage`的栈顶`next`指针指向的位置

> 这里只拿了一张page举例.

#### 1.15.2 小结

- 自动释放池是有N张`AutoreleasePoolPage`组成,每张page 4K大小, AutoreleasePoolPage是c++的类, AutoreleasePoolPage以双向链表连接起来形成一个自动释放池
- 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
- pop 时是传入边界对象(哨兵对象),然后对page 中的对象发送release 的消息

[自动释放池原理](https://www.jianshu.com/p/0afda1f23782) [AutoreleasePool底层实现原理](https://juejin.im/post/6844903609428115470)

### 1.16 ARC的实现原理？ARC下对retain, release做了哪些优化

ARC（自动引用计数）是苹果引入的一套机制，由编译器在编译期自动在合适的位置插入 retain、release、autorelease 等调用，帮助开发者管理对象的内存。

它的实现原理就是在编译层面插入相关代码,帮助补全MRC时代需要开发者手动填写的和管理的对象的相关内存操作的方法.

从代码编译成汇编的过程可以看到，编译器会插入引用计数相关调用，并结合 `isa` 指针信息做优化。

[理解 ARC 实现原理](https://juejin.im/post/6844903847622606861#heading-4)

下面通过 `isa` 的组成继续串联 ARC 优化逻辑。

isa的组成

```
union isa_t
{
    Class cls;
    uintptr_t bits;
    struct {
         uintptr_t nonpointer        : 1;//->表示使用优化的isa指针
         uintptr_t has_assoc         : 1;//->是否包含关联对象
         uintptr_t has_cxx_dtor      : 1;//->是否设置了析构函数，如果没有，释放对象更快
         uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 ->类的指针
         uintptr_t magic             : 6;//->固定值,用于判断是否完成初始化
         uintptr_t weakly_referenced : 1;//->对象是否被弱引用
         uintptr_t deallocating      : 1;//->对象是否正在销毁
         uintptr_t has_sidetable_rc  : 1;//1->在extra_rc存储引用计数将要溢出的时候,借助Sidetable(散列表)存储引用计数,has_sidetable_rc设置成1
        uintptr_t extra_rc          : 19;  //->存储引用计数
    };
};
```

其中`nonpointer`、`weakly_referenced`、`has_sidetable_rc`和`extra_rc`都是 `ARC`有直接关系的成员变量，其他的大多也有涉及到。

#### 1.16.1 retain,release做了哪些优化

大概可以分为如下

- TaggedPointer 指针优化
- !newisa.nonpointer：未优化的 isa 的情况下retain或者release
- newisa.nonpointer：已优化的 isa ， 这其中又分 extra_rc 溢出区别 我把相关代码站在下面并且把结论输出出来.

| 内存操作 | objc_retain | objc_release |
| --- | --- | --- |
| TaggedPointer | 值存在指针内，直接返回 | 直接返回 false。 |
| !nonpointer | 未优化的`isa`,使用`sidetable_retain()` | 未优化的`isa`执行`sidetable_release` |
| nonpointer | 已优化的`isa`,这其中又分`extra_rc`溢出和未溢出的两种情况 | 已优化的`isa`,分下溢和未下溢两种情况 |

| nonpointer已优化isa的extra_rc | objc_retain | objc_release |
| --- | --- | --- |
| 未溢出时 | `isa.extra_rc`+1 | NA |
| 溢出时 | 将`isa.extra_rc`中一半值转移至`sidetable`中,然后将`isa.has_sidetable_rc`设置为`true`,表示使用了`sidetable`来计算引用次数 | NA |
| 未下溢 | NA | extra_rc-- |
| 下溢 | NA | 从`sidetable`中借位给`extra_rc`达到半满,如果无法借位则说明引用计数归零需要进行释放,其中借位时可能保存失败会不断重试 |

> NA -> non available 不可获得

retain 源码如下：

```
ALWAYS_INLINE id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    if (isTaggedPointer()) return (id)this;     // 如果是 TaggedPointer 直接返回
    bool sideTableLocked = false;
    bool transcribeToSideTable = false;
    isa_t oldisa;
    isa_t newisa;
    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);  // 获取 isa
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);// 未优化的 isa 部分
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
            else return sidetable_retain();
        }
        if (slowpath(tryRetain && newisa.deallocating)) { // 正在被释放的处理
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            return nil;
        }
        // extra_rc 未溢出时引用计数++
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
        // extra_rc 溢出
        if (slowpath(carry)) {
            // newisa.extra_rc++ overflowed
            if (!handleOverflow) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);   // 重新调用该函数 入参 handleOverflow 为 true
            }
            // 保留一半引用计数,准备将另一半复制到 side table.
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
        //  更新 isa 值
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));
    if (slowpath(transcribeToSideTable)) {
        sidetable_addExtraRC_nolock(RC_HALF); // 将另一半复制到 side table side table.
    }
    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    return (id)this;
}
```

`release`源码

```
ALWAYS_INLINE bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    if (isTaggedPointer()) return false;
    bool sideTableLocked = false;
    isa_t oldisa;
    isa_t newisa;
 retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);// 未优化 isa
            if (sideTableLocked) sidetable_unlock();
            return sidetable_release(performDealloc);// 入参是否要执行 Dealloc 函数，如果为 true 则执行 SEL_dealloc
        }
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // donot ClearExclusive()
            goto underflow;
        }
        // 更新 isa 值
    } while (slowpath(!StoreReleaseExclusive(&isa.bits,
                                             oldisa.bits, newisa.bits)));
    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;
 underflow:
 	// 处理下溢，从 side table 中借位或者释放
    newisa = oldisa;
    if (slowpath(newisa.has_sidetable_rc)) { // 如果使用了 sidetable_rc
        if (!handleUnderflow) {
        	ClearExclusive(&isa.bits);// 调用本函数处理下溢
            return rootRelease_underflow(performDealloc);
        }
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF); // 从 sidetable 中借位引用计数给 extra_rc

        if (borrowed > 0) {
    	// extra_rc 是计算额外的引用计数，0 即表示被引用一次
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreReleaseExclusive(&isa.bits,
                                                oldisa.bits, newisa.bits);
            // 保存失败，恢复现场，重试
            if (!stored) {
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.nonpointer) {
                    uintptr_t overflow;
                    newisa2.bits =
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits,
                                                       newisa2.bits);
                    }
                }
            }
    	// 如果还是保存失败，则还回 side table
            if (!stored) {
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }
            sidetable_unlock();
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }
    // 没有使用 sidetable_rc ，或者 sidetable_rc 计数 == 0 的就直接释放
    // 如果已经是释放中，抛个过度释放错误
    if (slowpath(newisa.deallocating)) {
        ClearExclusive(&isa.bits);
        if (sideTableLocked) sidetable_unlock();
        return overrelease_error();
    }
    // 更新 isa 状态
    newisa.deallocating = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
    if (slowpath(sideTableLocked)) sidetable_unlock();
    // 执行 SEL_dealloc 事件
    __sync_synchronize();
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
}
```

#### 1.16.2 小结

到这里可以知道 引用计数分别保存在`isa.extra_rc`和`sidetable`中，当`isa.extra_rc`溢出时，将一半计数转移至`sidetable`中，而当其下溢时，又会将计数转回。当二者都为空时，会执行释放流程

### 1.17 ARC下哪些情况会造成内存泄漏

- block中的循环引用
- NSTimer的循环引用
- addObserver的循环引用
- delegate的强引用
- 大次数循环内存爆涨
- 非OC对象的内存处理（需手动释放）

### 1.18 `Method Swizzle`注意事项

1.  **需要注意的是交换方法实现后的副作用**, `method_exchangeImplementations()`.交换方法函数最终会以`objc_msgSend()`方式调用,副作用主要集中在第一个参数 如下示例

```
objc_msgSend(payment, @selector(quantity))
```

 方法交换后再去调用quantity方法将有可能会crash.解决这种副作用的方式是使用`method_setImplementation()`来替换原来的交换方式,这样才最为合理, 具体原理请参照 [Objc 黑科技 - Method Swizzle 的一些注意事项](https://www.ctolib.com/topics-103098.html)

2.  **避免交换父类方法**

    如果当前类没有实现被交换的方法且父类实现了,此时父类的实现会被交换,若此父类的多个继承者都在交换时会引起多次交换导致混乱,同时调用父类方法有可能因为找不到方法签名而crash.
    所以交换前都应该check能否为当前类添加被交换的函数的新的实现IMP,这个过程大概分为3步骤

    - `class_addMethod` check能否添加方法

```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)

```

> 给类cls的SEL添加一个实现IMP, 返回YES则表明类cls并未实现此方法，返回NO则表明类已实现了此方法。注意：添加成功与否，完全由该类本身来决定，与父类有无该方法无关。

- `class_replaceMethod` 替换类cls的SEL的函数实现为imp

```
class_replaceMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp,
                 const char * _Nullable types)

```

- `method_exchangeImplementations` 最终方法交换

```
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2)

```

3.  交换方法应在+load方法

这个前面讲消息转发的时候讲过,+load不是消息转发的方式实现的且在运行时初始化过程中类被加载的时候调用,而且父类,当前类,category,子类等 都会调用一次.所以这里最适合写方法交换的hook(Method Swizzle).

4.  交换的分类方法应该添加自定义前缀,避免冲突

    这个毫无疑问,方法名称一样的时候会出现,分类的方法会覆盖类中同名的方法.

[method swizzling你应该注意的点](https://blog.csdn.net/weixin_34168700/article/details/88762738)

### 1.19 atomic 的内部实现与线程安全

#### 1.19.1 atomic内部实现

```
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    ...
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    return objc_autoreleaseReturnValue(value);
}
```

```
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    ...
    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;
        slotlock.unlock();
    }
    objc_release(oldValue);
}
```

`property` 的 `atomic` 是采用 `spinlock_t`自旋锁实现的.

#### 1.19.2 线程安全边界

`atomic`通过这种方法.在运行时仅仅是保证了`set`,`get`方法的原子性.所以使用atomic并不能保证线程安全。

### 1.20 iOS 中内省的几个方法有哪些？内部实现原理是什么?

首先要理解 `introspection` 这个词，即"内省"，在 iOS 开发中我们也常把这类能力称为反射。

内省方法 例如常用的`NSObject`中的`isKindOfClass:` 通过实例对象判断`class`这就是一种内省方法或者叫反射方法,但我认为`NSClassFromString()`这个应该也算一种反射方法.

#### 1.20.1 iOS 中内省的几个方法

我们从NSObject.h中看下吧

```
- (BOOL)isKindOfClass:(Class)aClass; //判断是否是这个类或者这个类的子类的实例
- (BOOL)isMemberOfClass:(Class)aClass; //判断是否是这个类的实例
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;  //判断是否遵守某个协议
+ (BOOL)conformsToProtocol:(Protocol *)protocol; //判断某个类是否遵守某个协议
- (BOOL)respondsToSelector:(SEL)aSelector;  //判读实例是否有这样方法
+ (BOOL)instancesRespondToSelector:(SEL)aSelector; //判断类是否有这个方法
...
```

#### 1.20.2 内部实现原理

1. `isKindOfClass:`

```
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = self->ISA(); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
    }

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
    }
```

从源码可见，二者都是沿继承链逐级比对：类方法从自身的元类（`self->ISA()`）出发，实例方法从对象所属的类（`[self class]`）出发，再沿 `superclass` 链逐级向上，只要链上任一节点等于传入的 `cls`，即返回 YES。区别仅在于起点不同——类方法比对的是元类继承链，实例方法比对的是类继承链。

2. `isMemberOfClass:`

```
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
    }

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
    }
```

这两个方法都只做一次精确比对，且不沿继承链向上查找：类方法判断 `self->ISA()`（元类）是否等于 cls，实例方法判断 `[self class]` 是否等于 cls。

3. `conformsToProtocol:`

```
+ (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = self; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
    }

- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
    }
```

两个方法最终还是去isa->data()->protocols 拿到相关协议然后判断是否存在相关协议 如下代码：

```
BOOL class_conformsToProtocol(Class cls, Protocol *proto_gen)
{
    protocol_t *proto = newprotocol(proto_gen);
    if (!cls) return NO;
    if (!proto_gen) return NO;
    mutex_locker_t lock(runtimeLock);
    checkIsKnownClass(cls);
    ASSERT(cls->isRealized())
    for (const auto& proto_ref : cls->data()->protocols) {
        protocol_t *p = remapProtocol(proto_ref);
        if (p == proto || protocol_conformsToProtocol_nolock(p, proto)) {
            return YES;
        }
    }
    return NO;
}
```

> 这里可以清晰的看到for循环 取出相关protocol指针 然后通过指针和传入的参数生成的`proto`对比

4. `respondsToSelector:`

```
+ (BOOL)respondsToSelector:(SEL)sel {
    return class_respondsToSelector_inst(self, sel, self->ISA());
    }

- (BOOL)respondsToSelector:(SEL)sel {
    return class_respondsToSelector_inst(self, sel, [self class]);
    }
```

这个源码比较麻烦 我简单叙述一下吧 实际上调用栈比较深就是一直寻找到当前实例能响应哪些方法,当前类没有就去父类,父类没有则直到元类.

```
respondsToSelector:
	|__ class_respondsToSelector_inst()
		|__ lookUpImpOrNil()
			|__ lookUpImpOrForward()
				返回IMP结果
```

这其实是方法查找（`lookUpImpOrForward`）的过程，会沿 类 → 父类 → … 直至根类的继承链查找 IMP；它发生在消息转发之前，相关内容可回看前文的消息发送与查找部分。

以上列举了一些常用的内省方法，其余方法原理大同小异，本质都是借助 isa 找到类对象，再读取其中存储的方法、协议、属性等信息后返回结果。

### 1.21 `class`、`objc_getClass`、`object_getClass` 对比

我用 Xcode 建了一个 demo，分别打印 ViewController 三种方式的结果：

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    Class cls1 = [self class];
    Class cls2 = object_getClass(cls1);
    Class cls3 = objc_getClass(object_getClassName([self class]));
    NSLog(@"%p",cls1);
    NSLog(@"%p",cls2);
    NSLog(@"%p",cls3);
    }
    @end
```

输出

```
2020-08-31 16:15:48.150285+0800 ClassDemo[5582:55836] 0x10205b3b0
2020-08-31 16:15:48.150456+0800 ClassDemo[5582:55836] 0x10205b3d8
2020-08-31 16:15:48.150575+0800 ClassDemo[5582:55836] 0x10205b3b0
```

我简单列举了一张表格

|  | `class` | `object_getClass()` | `objc_getClass()` |
| --- | --- | --- | --- |
| 传入参数 | N/a | id 类型 | 类名字符串 |
| 返回内容 | 接收者所属的类 | 该 id 的 isa 指针所指向的类 | 由类名查到的类对象 |
| 接收者为实例对象时 | 与 `object_getClass()` 一致，返回其所属类 | 返回该实例的类对象 | N/a |
| 接收者为类对象时 | 返回类对象自身（self） | 返回该类的元类（meta-class） | N/a |

> 原因：因为class返回的是self，而object_getClass返回的是isa指向的对象

---

<a id="第二章runloop-与-kvo"></a>
## 第二章 RunLoop 与 KVO


### 2.1 Runloop 和线程的关系？

- 一个线程对应一个 Runloop。

- 主线程的默认就有了 Runloop。

- 子线程的 Runloop 以懒加载的形式创建。

- Runloop 存储在一个全局的可变字典里，线程是 key ，Runloop 是 value。

### 2.2 RunLoop的运行模式

- RunLoop的运行模式共有5种，RunLoop只会运行在一个模式下，要切换模式，就要暂停当前模式，重新启动一个运行模式

```
    - kCFRunLoopDefaultMode, App的默认运行模式，通常主线程是在这个运行模式下运行
    - UITrackingRunLoopMode, 跟踪用户交互事件（用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他Mode影响）
    - kCFRunLoopCommonModes, 伪模式，不是一种真正的运行模式
    - UIInitializationRunLoopMode：在刚启动App时第进入的第一个Mode，启动完成后就不再使用
    - GSEventReceiveRunLoopMode：接受系统内部事件，通常用不到
    
    ```

### 2.3 runloop内部逻辑？

- 实际上 RunLoop 就是这样一个函数，其内部是一个 do-while 循环。当你调用 CFRunLoopRun() 时，线程就会一直停留在这个循环里；直到超时或被手动停止，该函数才会返回。

    ![RunLoop](https://upload-images.jianshu.io/upload_images/17495317-90f472cc5d134bdb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 内部逻辑：

    1.  通知 Observer 已经进入了 RunLoop

    2.  通知 Observer 即将处理 Timer

    3.  通知 Observer 即将处理非基于端口的输入源（即将处理 Source0）

    4.  处理那些准备好的非基于端口的输入源（处理 Source0）

    5.  如果基于端口的输入源准备就绪并等待处理，请立刻处理该事件。转到第 9 步（处理 Source1）

    6.  通知 Observer 线程即将休眠

    7.  将线程置于休眠状态，直到发生以下事件之一

        - 事件到达基于端口的输入源（port-based input sources，即 Source1）

        - Timer 到时间执行

        - 外部手动唤醒

        - 为 RunLoop 设定的时间超时

    8.  通知 Observer 线程刚被唤醒（还没处理事件）

    9.  处理待处理事件

        - 如果是 Timer 事件，处理 Timer 并重新启动循环，跳到第 2 步

        - 如果输入源被触发，处理该事件（文档上是 deliver the event）

        - 如果 RunLoop 被手动唤醒但尚未超时，重新启动循环，跳到第 2 步

### 2.4 autoreleasePool 在何时被释放？

- App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

- 第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是 -2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

- 第二个 Observer 监视了两个事件：BeforeWaiting(即将进入休眠) 时，先调用 _objc_autoreleasePoolPop() 释放上一轮循环产生的旧池，再调用 _objc_autoreleasePoolPush() 创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 释放当前自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

- 在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显式创建 Pool 了。

### 2.5 GCD 在Runloop中的使用？

- GCD由 子线程 返回到 主线程,只有在这种情况下才会触发 RunLoop。会触发 RunLoop 的 Source 1 事件。

### 2.6 AFNetworking 中如何运用 Runloop?

- AFURLConnectionOperation 这个类是基于 NSURLConnection 构建的，其希望能在后台线程接收 Delegate 回调。为此 AFNetworking 单独创建了一个线程，并在这个线程中启动了一个 RunLoop：

    ```
    + (void)networkRequestThreadEntryPoint:(id)__unused object {
        @autoreleasepool {
            [[NSThread currentThread] setName:@"AFNetworking"];
            NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
            [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
            [runLoop run];
        }
    }
    
    + (NSThread *)networkRequestThread {
        static NSThread *_networkRequestThread = nil;
        static dispatch_once_t oncePredicate;
        dispatch_once(&oncePredicate, ^{
            _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
            [_networkRequestThread start];
        });
        return _networkRequestThread;
    }
    
    ```


- RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 AFNetworking 在 [runLoop run] 之前先创建了一个新的 NSMachPort 添加进去了。通常情况下，调用者需要持有这个 NSMachPort (mach_port) 并在外部线程通过这个 port 发送消息到 loop 内；但此处添加 port 只是为了让 RunLoop 不至于退出，并没有用于实际的发送消息。

    ```
    - (void)start {
        [self.lock lock];
        if ([self isCancelled]) {
            [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
        } else if ([self isReady]) {
            self.state = AFOperationExecutingState;
            [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
        }
        [self.lock unlock];
    }

    ```

- 当需要这个后台线程执行任务时，AFNetworking 通过调用 [NSObject performSelector:onThread:..] 将这个任务扔到了后台线程的 RunLoop 中。

### 2.7 PerformSelector 的实现原理？

- 当调用 NSObject 的 performSelector:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

- 当调用 performSelector:onThread: 时，会向目标线程的 RunLoop 投递一个 Source0 任务来触发（并不创建 Timer）；同样地，如果目标线程没有运行 RunLoop，该方法也无法被执行。

### 2.8 PerformSelector:afterDelay:这个方法在子线程中是否起作用？

- 不起作用，子线程默认没有 Runloop，也就没有 Timer。可以使用 GCD的dispatch_after来实现

### 2.9 事件响应的过程？

- 苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

- 当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考这里。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的 App 进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

- _UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegan/Moved/Ended/Cancelled 事件都是在这个回调中完成的。

### 2.10 手势识别的过程？

- 当 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegan/Moved/Ended 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

- 苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个 Observer 的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer 的回调。

- 当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

### 2.11 CADisplayLink 和 NSTimer 哪个更精确？

CADisplayLink 更精确

- iOS设备的屏幕刷新频率是固定的，CADisplayLink在正常情况下会在每次刷新结束都被调用，精确度相当高。

- NSTimer 的精确度就显得低一些：当 NSTimer 的触发时间到达时，如果 RunLoop 正处于阻塞状态，触发就会被推迟到下一个 RunLoop 周期。此外，NSTimer 提供了 tolerance 属性，可设置允许的触发时间误差范围，系统会在该范围内调整触发时机以优化性能（这是放宽精度、而非提高精度）。

- CADisplayLink使用场合相对专一，适合做UI的不停重绘，比如自定义动画引擎或者视频播放的渲染。NSTimer的使用范围要广泛的多，各种需要单次或者循环定时处理的任务都可以使用。在UI相关的动画或者显示内容使用 CADisplayLink比起用NSTimer的好处就是我们不需要在格外关心屏幕的刷新频率了，因为它本身就是跟屏幕刷新同步的。


### 2.12 Runloop

#### 2.12.1 app如何接收到触摸事件的

[iOS触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

[![](https://upload-images.jianshu.io/upload_images/13277235-f667a2cf8887ce98.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200902iOSinterviewAnswers/runloop_event_receive.jpg)

通过上图可以看出整个流程就是 我们app启动默认会通过machPort监听端口的方式 来接受IOHIDEvent 来接收和处理触摸事件.

#### 2.12.2 为什么只有主线程的runloop是开启的

main()函数中调用UIApplicationMain，这里会创建一个主线程，用于UI处理，为了让程序可以一直运行并接收事件，所以在主线程中开启一个runloop，让主线程常驻.

#### 2.12.3 为什么只在主线程刷新UI

我们所有用到的UI都是来自于UIKit这个基础库.因为objc不是一门线程安全的语言所以存在多线程读写不同步的问题,如果使用加锁的方式操作系统开销很大,会耗费大量的系统资源(内存、时间片轮转、cpu处理速度...)，加上上面讲到的系统事件的接收处理都在主线程,如果UI异步线程的话 还会存在 同步处理事件的问题,所以多点触摸手势等一些事件要保持和UI在同一个线程相对是最优解.

另一方面，屏幕的刷新率通常为 60Hz（即每秒刷新 60 次，iPad Pro 等设备为 120Hz）；理想情况下主线程会以相应频率配合刷新，如此高频的处理是为了保证屏幕图像垂直同步、不卡顿。如果把 UI 放到异步线程，很难保证这一过程的同步更新；即便能够保证，相对主线程而言，额外的系统资源开销、线程调度等也会占用大量资源，与"在同一线程专门做一件事"相比反而得不偿失。

#### 2.12.4 PerformSelector和runloop的关系

当调用NSObject的 performSelector:相关的时候,内部会创建一个timer定时器添加到当前线程的runloop中,如果当前线程没有启动runloop,则该方法不会被调用.

开发中遇到最多的问题就是这个performSelector: 导致对象的延迟释放,这里开发过程中注意一下,可以用单次的NSTimer替代.

详细可以参考[Runloop与performSelector](https://juejin.im/post/6844903781755256840)

#### 2.12.5 如何使线程保活？

想要线程保活的话就开启该线程的runloop即可,注意:在NSThread执行的方法中添加while(true){}，这样是模拟runloop的运行原理，结合GCD的信号量，在{}代码块中处理任务.

但是注意 开启runloop的方法要正确

如下代码

```
//测试开启线程
- (void)memoryTest {
    for (int i = 0; i < 100000; ++i) {
        NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [thread start];
        [self performSelector:@selector(stopThread) onThread:thread withObject:nil waitUntilDone:YES];
    }
}
//线程停止
- (void)stopThread {
    CFRunLoopStop(CFRunLoopGetCurrent());
    NSThread *thread = [NSThread currentThread];
    [thread cancel];
}
//运行线程的runloop 注意 意添加的那个空port,否则会出现内存泄露
- (void)run {
    @autoreleasepool {
        NSLog(@"current thread = %@", [NSThread currentThread]);
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        if (!self.emptyPort) {
            self.emptyPort = [NSMachPort port];
        }
        [runLoop addPort:self.emptyPort forMode:NSDefaultRunLoopMode];
        [runLoop runMode:NSRunLoopCommonModes beforeDate:[NSDate distantFuture]];
    }
}
//下列代码用于模拟线程内部做的一些耗时任务
- (void)printSomething {
    NSLog(@"current thread = %@", [NSThread currentThread]);
    [self performSelector:@selector(printSomething) withObject:nil afterDelay:1];
}
//模拟手动点击按钮 让 runloop停掉
- (void)stopButtonDidClicked:(id)sender {
    [self performSelector:@selector(stopRunloop) onThread:self.thread withObject:nil waitUntilDone:YES];
}

- (void)stopRunloop {
    CFRunLoopStop(CFRunLoopGetCurrent());
}
```

参考：[iOS开发深入研究Runloop与线程保活](https://allluckly.cn/%E6%8A%95%E7%A8%BF/tuogao55)

### 2.13 KVO

#### 2.13.1 KVO的实现原理

KVO 的核心是 **isa-swizzling**：当对某对象的属性添加观察后，runtime 会动态派生一个子类，重写被观察属性的 setter，在赋值前后分别调用 `willChangeValueForKey:` 与 `didChangeValueForKey:`，后者会触发观察者的 `observeValueForKeyPath:ofObject:change:context:` 回调，从而实现属性变化前后的通知。

派生出的子类命名格式为 `NSKVONotifying_类名`（例如 NSKVONotifying_Person）。随后 runtime 会把被观察对象的 isa 指针指向这个子类（即 isa-swizzling）；同时该子类重写了 `class` 方法，使外部调用 `-class` 时仍返回原始类，从而对使用者透明。

下面示例代码为Person类的name添加KVO的模拟实验

```
// 以下为示意性伪代码，用于说明派生子类对 setter 的改写逻辑
- (void)setName:(NSString *)name {
    _NSSetNameValueAndNotify(self, _cmd, name);
}

void _NSSetNameValueAndNotify(id self, SEL _cmd, NSString *name) {
    [self willChangeValueForKey:@"name"];
    [super setName:name];   // 调用父类（原始类）的实现完成真正赋值
    [self didChangeValueForKey:@"name"];
}

- (void)didChangeValueForKey:(NSString *)key {
    // 通知所有观察者
    [observer observeValueForKeyPath:key ofObject:self change:nil context:nil];
}
```

问题来了如何动态创建类呢?

```
//动态创建XXCustomClass
Class customClass = objc_allocateClassPair([NSObject class], "XXCustomClass", 0);
// 添加实例变量
class_addIvar(customClass, "age", sizeof(int), 0, "i");
// 动态添加方法
class_addMethod(customClass, @selector(hahahha), (IMP)hahahha, "v@:");

//需要实现的方法
void hahahha(id self, SEL _cmd)
{
    NSLog(@"hahahha====");
}

- (void)hahahha{

}

//最后注册到运行时环境
objc_registerClassPair(customClass);
```

> [类型编码 v@: 表示方法的返回值与参数](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

具体原理以及自定义实现KVO可以参考[KVO详解及底层实现](https://cloud.tencent.com/developer/article/1136759)

#### 2.13.2 如何手动关闭KVO?

被观察的对象复写如下方法 返回`NO`即可关闭KVO

```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
	return NO;
}
```

如果关闭后还想触发 KVO的话 修改需要手动调用在变量setter的前后 主动调用 `willChangeValueForKey:`和`didChangeValueForKey:`

#### 2.13.3 通过KVC修改属性会触发KVO么?

会的

#### 2.13.4 哪些情况下使用kvo会崩溃，怎么防护崩溃？

使用不当会导致崩溃，常见场景包括：

- 添加与移除未成对出现：重复移除观察者，或移除未注册的观察者，会抛出 `NSRangeException`。
- 观察者已释放但未移除：对象释放后仍收到 KVO 回调，造成野指针崩溃（常发生在 dealloc 时或对象销毁前未正确移除 Observer）。
- 多线程下并发添加 / 移除观察者。
- 观察了一个对象上并不存在的 keyPath。

如何防护？

1. 保证 `addObserver` 与 `removeObserver` 成对匹配，不重复移除、不移除未注册的观察者。
2. 一定要在对象销毁前移除观察者，避免野指针崩溃。
3. 可借助第三方库（如 Facebook 的 KVOController / FBKVOController）管理 KVO，其内部会在合适时机自动移除观察者，从而避免崩溃。

#### 2.13.5 KVO的优缺点

优点:

- 方便两个对象间同步状态(keypath)更加方便,一般都是在A类要观察B类的属性的变化.
- 非侵入式的得到某内部对象的状态改变并作出响应.(就是在不改变原来对象类的代码情况下即可做出对该对象的状态变化进行监听)
- 可以分别捕获属性变更前、后两个时机的状态。
- 可以通过 keyPath 对嵌套对象的属性进行监听。

缺点:

- 需要手动移除观察者,不移除容易造成crash.
- 注册与移除必须成对匹配出现。
- keyPath 参数是字符串类型，若对象的属性名在重构中被修改，字符串不会被编译器检查出来，从而引发运行期错误。
- 观察是通过重写 NSObject 的 KVO 相关方法实现的，若能改为面向协议（protocol）的方式会更优雅。

---

<a id="第三章内存管理"></a>
## 第三章 内存管理


### 3.1 什么情况使用weak关键字，相比assign有什么不同？

- 什么情况使用 weak 关键字？

    在 ARC 中,在有可能出现循环引用的时候,往往要通过让其中一端使用 weak 来解决,比如: delegate 代理属性

    自身已经对它进行一次强引用,没有必要再强引用一次,此时也会使用 weak,自定义 IBOutlet 控件属性一般也使用 weak；当然，也可以使用strong。在下文也有论述：《IBOutlet连出来的视图属性为什么可以被设置成weak?》

- 不同点：

    weak 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。 而 assign 的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSInteger 等)的简单赋值操作。

    assign 可以用非 OC 对象,而 weak 必须用于 OC 对象

### 3.2 如何让自己的类用copy修饰符？如何重写带copy关键字的setter？

- 若想令自己所写的对象具有拷贝功能，则需实现 NSCopying 协议。如果自定义的对象分为可变版本与不可变版本，那么就要同时实现 NSCopying 与 NSMutableCopying 协议。

    具体步骤：

    需声明该类遵从 NSCopying 协议

    实现 NSCopying 协议。该协议只有一个方法:

    ```
    - (id)copyWithZone:(NSZone *)zone;

    ```

    注意：一提到让自己的类用 copy 修饰符，我们总是想覆写copy方法，其实真正需要实现的却是 “copyWithZone” 方法。

- 重写带 copy 关键字的 setter，例如：

    ```
    - (void)setName:(NSString *)name {
        //[_name release];
        _name = [name copy];
    }
    
    ```

### 3.3 深拷贝与浅拷贝

浅拷贝只是对指针的拷贝，拷贝后两个指针指向同一个内存空间，深拷贝不但对指针进行拷贝，而且对指针指向的内容进行拷贝，经深拷贝后的指针是指向两个不同地址的指针。

在 Objective-C 中，对象作为参数或返回值传递的都是指针，并不会自动触发拷贝；是否产生副本，取决于是否显式调用 `copy` / `mutableCopy`。这两个方法的拷贝深浅，要按"源对象是否可变"来区分：

- `copy` 方法：对不可变对象（如 NSString、NSArray）执行 copy 是**浅拷贝**——仅返回原对象的强引用，不产生新对象；对可变对象（如 NSMutableArray）执行 copy 是**深拷贝**——产生一个新的不可变副本。

- `mutableCopy` 方法：无论源对象是否可变，都会产生一个新的可变对象（属于单层深拷贝）。但对容器类而言，新容器内部的元素仍是指针拷贝（与源容器共享同一批元素对象），并非递归的完全深拷贝。

### 3.4 @property的本质是什么？ivar、getter、setter是如何生成并添加到这个类中的

- @property 的本质是实例变量（ivar）+存取方法（access method ＝ getter + setter）,即 @property = ivar + getter + setter;

    “属性” (property)作为 Objective-C 的一项特性，主要的作用就在于封装对象中的数据。 Objective-C 对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过“存取方法”(access method)来访问。其中，“获取方法” (getter)用于读取变量值，而“设置方法” (setter)用于写入变量值。

- ivar、getter、setter 是自动合成这个类中的

    完成属性定义后，编译器会自动编写访问这些属性所需的方法，此过程叫做“自动合成”(autosynthesis)。需要强调的是，这个过程由编译器在编译期执行，所以源代码里看不到这些“合成方法”(synthesized method)。除了生成方法代码 getter、setter 之外，编译器还要自动向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。在前例中，会生成两个实例变量，其名称分别为 _firstName 与 _lastName。也可以在类的实现代码里通过 @synthesize 语法来指定实例变量的名字.

### 3.5 @protocol和category中如何使用@property

- 在 protocol 中使用 property 只会生成 setter 和 getter 方法声明,我们使用属性的目的,是希望遵守我协议的对象能实现该属性

- category 使用 @property 也是只会生成 setter 和 getter 方法的声明,如果我们真的需要给 category 增加属性的实现,需要借助于运行时的两个函数：objc_setAssociatedObject和objc_getAssociatedObject

### 3.6 简要说一下 @autoreleasepool 的数据结构？

其底层是由若干个 `AutoreleasePoolPage` 节点组成的双向链表，相邻节点通过 `parent` 与 `child` 指针相连（并非环形）。

每次 push 一个自动释放池，都会向当前页压入一个哨兵对象（`POOL_BOUNDARY`）作为边界标记。

每个 `AutoreleasePoolPage` 都有一个 `next` 指针，指向该页中下一个可用的栈位置；当一页存满后，会创建新的 page 并通过 `child` 指针链接，`next` 转而指向新页继续存储。

### 3.7 EXC_BAD_ACCESS在什么情况下出现？

访问了悬垂指针（野指针），例如对一个已经释放的对象执行 release、访问其成员变量或向其发送消息；此外，无限递归导致栈溢出也可能触发 EXC_BAD_ACCESS。

### 3.8 使用CADisplayLink、NSTimer有什么注意点？

CADisplayLink、NSTimer会造成循环引用，可以使用YYWeakProxy或者为CADisplayLink、NSTimer添加block方法解决循环引用

### 3.9 iOS内存分区情况

- 栈区（Stack）

    由编译器自动分配释放，存放函数的参数，局部变量的值等

    栈是向低地址扩展的数据结构，是一块连续的内存区域

- 堆区（Heap）

    由程序员分配释放

    是向高地址扩展的数据结构，是不连续的内存区域

- 全局区

    全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域，未初始化的全局变量和未初始化的静态变量在相邻的另一块区域

    程序结束后由系统释放

- 常量区

    常量字符串就是放在这里的

    程序结束后由系统释放

- 代码区

    存放函数体的二进制代码

- 注：

    - 在 iOS 中，堆区的内存是应用程序共享的，堆中的内存分配是系统负责的

    - 系统使用一个链表来维护所有已经分配的内存空间（系统仅仅记录，并不管理具体的内容）

    - 变量使用结束后，需要释放内存，OC 中是判断引用计数是否为 0，如果是就说明没有任何变量使用该空间，那么系统将其回收

    - 当一个 app 启动后，代码区、常量区、全局区大小就已经固定，因此指向这些区的指针不会产生崩溃性的错误。而堆区和栈区是时时刻刻变化的（堆的创建销毁，栈的弹入弹出），所以当使用一个指针指向这个区里面的内存时，一定要注意内存是否已经被释放，否则会产生程序崩溃（也即是野指针报错）

### 3.10 iOS内存管理方式

- Tagged Pointer（小对象）

    Tagged Pointer 专门用来存储小的对象，例如 NSNumber 和 NSDate

    Tagged Pointer 指针的值不再是地址了，而是真正的值。所以，实际上它不再是一个对象了，它只是一个披着对象皮的普通变量而已。所以，它的内存并不存储在堆中，也不需要 malloc 和 free

    在内存读取上有着 3 倍的效率，创建时比以前快 106 倍

    objc_msgSend 能识别 Tagged Pointer，比如 NSNumber 的 intValue 方法，直接从指针提取数据

    使用 Tagged Pointer 后，指针内存储的数据变成了 Tag + Data，也就是将数据直接存储在了指针中

- NONPOINTER_ISA （指针中存放与该对象内存相关的信息） 苹果将 isa 设计成了联合体，在 isa 中存储了与该对象相关的一些内存的信息，原因也如上面所说，并不需要 64 个二进制位全部都用来存储指针。

    isa 的结构：

    ```
    // x86_64 架构
    struct {
        uintptr_t nonpointer        : 1;  // 0:普通指针，1:优化过，使用位域存储更多信息
        uintptr_t has_assoc         : 1;  // 对象是否含有或曾经含有关联引用
        uintptr_t has_cxx_dtor      : 1;  // 表示是否有C++析构函数或OC的dealloc
        uintptr_t shiftcls          : 44; // 存放着 Class、Meta-Class 对象的内存地址信息
        uintptr_t magic             : 6;  // 用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1;  // 是否被弱引用指向
        uintptr_t deallocating      : 1;  // 对象是否正在释放
        uintptr_t has_sidetable_rc  : 1;  // 是否需要使用 sidetable 来存储引用计数
        uintptr_t extra_rc          : 8;  // 引用计数能够用 8 个二进制位存储时，直接存储在这里
    };

    // arm64 架构
    struct {
        uintptr_t nonpointer        : 1;  // 0:普通指针，1:优化过，使用位域存储更多信息
        uintptr_t has_assoc         : 1;  // 对象是否含有或曾经含有关联引用
        uintptr_t has_cxx_dtor      : 1;  // 表示是否有C++析构函数或OC的dealloc
        uintptr_t shiftcls          : 33; // 存放着 Class、Meta-Class 对象的内存地址信息
        uintptr_t magic             : 6;  // 用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1;  // 是否被弱引用指向
        uintptr_t deallocating      : 1;  // 对象是否正在释放
        uintptr_t has_sidetable_rc  : 1;  // 是否需要使用 sidetable 来存储引用计数
        uintptr_t extra_rc          : 19;  // 引用计数能够用 19 个二进制位存储时，直接存储在这里
    };

    ```

    这里的 has_sidetable_rc 和 extra_rc，has_sidetable_rc 表明该指针是否引用了 sidetable 散列表，之所以有这个选项，是因为少量的引用计数是不会直接存放在 SideTables 表中的，对象的引用计数会先存放在 extra_rc 中，当其被存满时，才会存入相应的 SideTables 散列表中，SideTables 中有很多张 SideTable，每个 SideTable 也都是一个散列表，而引用计数表就包含在 SideTable 之中。

- 散列表（引用计数表、弱引用表）

    引用计数要么存放在 isa 的 extra_rc 中，要么存放在引用计数表中，而引用计数表包含在一个叫 SideTable 的结构中，它是一个散列表，也就是哈希表。而 SideTable 又包含在一个全局的 StripedMap 的哈希映射表中，这个表的名字叫 SideTables。

    当一个对象访问 SideTables 时：

    - 首先会取得对象的地址，将地址进行哈希运算，与 SideTables 中 SideTable 的个数取余，最后得到的结果就是该对象所要访问的 SideTable

    - 在取得的 SideTable 中的 RefcountMap 表中再进行一次哈希查找，找到该对象在引用计数表中对应的位置

    - 如果该位置存在对应的引用计数，则对其进行操作，如果没有对应的引用计数，则创建一个对应的 size_t 对象，其实就是一个 uint 类型的无符号整型

    弱引用表也是一张哈希表的结构，其内部包含了每个对象对应的弱引用表 weak_entry_t，而 weak_entry_t 是一个结构体数组，其中包含的则是每一个对象弱引用的对象所对应的弱引用指针。

### 3.11 循环引用

#### 3.11.1 概述

iOS内存中的分区有：堆、栈、静态区。其中，栈和静态区是操作系统自己管理回收，不会造成循环引用。在堆中的相互引用无法回收，有可能造成循环引用。

> 循环引用的实质：多个对象相互之间存在强引用，导致彼此无法释放、系统无法回收。

> 解决循环引用一般是将 strong 引用改为 weak 引用。

#### 3.11.2 循环引用场景分析及解决方法

##### 3.11.2.1 视图容器与其子视图（如 UITableView 与 Cell）

> 如：在使用UITableView 的时候，将 UITableView 给 Cell 使用，cell 中的 strong 引用会造成循环引用。

```
// controller
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    TestTableViewCell *cell =[tableView dequeueReusableCellWithIdentifier:@"UITableViewCellId" forIndexPath:indexPath];
    cell.tableView = tableView;
    return cell;
}

// cell
@interface TestTableViewCell : UITableViewCell
@property (nonatomic, strong) UITableView *tableView; // strong 造成循环引用
@end

```

> 解决：strong 改为 weak

```
// cell
@interface TestTableViewCell : UITableViewCell
@property (nonatomic, weak) UITableView *tableView; // strong 改为 weak
@end

```

##### 3.11.2.2 block

> block在copy时都会对block内部用到的对象进行强引用的。

```
self.testObject.testCircleBlock = ^{
   [self doSomething];
};

```

self将block作为自己的属性变量，而在block的方法体里面又引用了 self 本身，此时就很简单的形成了一个循环引用。

应该将 self 改为弱引用

```
__weak typeof(self) weakSelf = self;
 self.testObject.testCircleBlock = ^{
      __strong typeof (weakSelf) strongSelf = weakSelf;
      [strongSelf doSomething];
};

```

> 在 ARC 中，在被拷贝的 block 中无论是直接引用 self 还是通过引用 self 的成员变量间接引用 self，该 block 都会 retain self。

- **快速定义宏**

```
    // weak obj
    #define WEAK_OBJ(type)  __weak typeof(type) weak##type = type;

    // strong obj
    /#define STRONG_OBJ(type)  __strong typeof(type) str##type = weak##type;

```

##### 3.11.2.3 Delegate

delegate 属性的声明如下：

```
@property (nonatomic, weak) id <TestDelegate> delegate;

```

如果将 weak 改为 strong，则会造成循环引用

```
// self -> AViewController
BViewController *bVc = [BViewController new];
bVc.delegate = self;   // 让 bVc 把 AViewController 作为自己的 delegate
[self.navigationController pushViewController:bVc animated:YES];

   // 假如 delegate 是 strong 的情况：
   // AViewController 持有（push）了 bVc            ===> bVc 引用计数 +1
   // bVc.delegate 又强引用了 AViewController       ===> AViewController 引用计数 +1
   // 结果：AViewController 与 bVc 相互强引用，形成循环引用

```

##### 3.11.2.4 NSTimer

NSTimer 的 target 对传入的参数都是强引用（即使是 weak 对象）

![](https://upload-images.jianshu.io/upload_images/17495317-212294bb1b4e00de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决办法: 《Effective Objective-C 》中的52条方法

```
#import <Foundation/Foundation.h>

@interface NSTimer (YPQBlocksSupport)

+ (NSTimer *)ypq_scheduledTimeWithTimeInterval:(NSTimeInterval)interval
                                         block:(void(^)())block
                                       repeats:(BOOL)repeats;

@end

#import "NSTimer+YPQBlocksSupport.h"

@implementation NSTimer (YPQBlocksSupport)

+ (NSTimer *)ypq_scheduledTimeWithTimeInterval:(NSTimeInterval)interval
                                         block:(void(^)())block
                                       repeats:(BOOL)repeats
{
    return [self scheduledTimerWithTimeInterval:interval
                                         target:self
                                       selector:@selector(ypq_blockInvoke:) userInfo:[block copy]
                                        repeats:repeats];
}

- (void)ypq_blockInvoke:(NSTimer *)timer
{
    void (^block)() = timer.userInfo;
    if(block)
    {
        block();
    }
}

@end

```

使用方式：

```
__weak ViewController * weakSelf = self;
[NSTimer ypq_scheduledTimeWithTimeInterval:4.0f
                                     block:^{
                                         ViewController * strongSelf = weakSelf;
                                         [strongSelf afterThreeSecondBeginAction];
                                     }
                                   repeats:YES];

```

> 计时器保留其目标对象，反复执行任务导致的循环，确实要注意，另外在dealloc的时候，不要忘了调用计时器中的 invalidate方法。


---

<a id="第四章多线程"></a>
## 第四章 多线程


### 4.1 进程与线程

- 进程：

    1. 进程是一个具有一定独立功能的程序关于某次数据集合的一次运行活动，它是操作系统分配资源的基本单元.

    2. 进程是指在系统中正在运行的一个应用程序，就是一段程序的执行过程,我们可以理解为手机上的一个app.

    3. 每个进程之间是独立的，每个进程均运行在其专用且受保护的内存空间内，拥有独立运行所需的全部资源

- 线程

    1. 程序执行流的最小单元，线程是进程中的一个实体.

    2. 一个进程要想执行任务,必须至少有一条线程.应用程序启动的时候，系统会默认开启一条线程,也就是主线程

- 进程和线程的关系

    1. 线程是进程的执行单元，进程的所有任务都在线程中执行

    2. 线程是 CPU 调度（执行）的最小单位

    3. 一个程序可以对应多个进程(多进程),一个进程中可有多个线程,但至少要有一条线程

    4. 同一个进程内的线程共享进程资源

### 4.2 什么是多线程？

- 多线程的实现原理：事实上，同一时间内单核的CPU只能执行一个线程，多线程是CPU快速的在多个线程之间进行切换（调度），造成了多个线程同时执行的假象。

- 如果是多核CPU就真的可以同时处理多个线程了。

- 多线程的目的是为了并发执行多项任务，通过提高 CPU 等资源的利用率来提升程序的整体效率。

### 4.3 多线程的优点和缺点

- 优点:

    能适当提高程序的执行效率

    能适当提高资源利用率（CPU、内存利用率）

- 缺点:

    开启线程需要占用一定的内存空间（默认情况下，主线程占用1M，子线程占用512KB），如果开启大量的线程，会占用大量的内存空间，降低程序的性能

    线程越多，CPU在调度线程上的开销就越大

    程序设计更加复杂：比如线程之间的通信、多线程的数据共享

### 4.4 多线程的 并行 和 并发 有什么区别？

- 并发（Concurrency）：指在同一时间段内处理多个任务。在单核上通过时间片快速切换交替执行，宏观上像是同时进行，微观上仍是顺序执行。

- 并行（Parallelism）：指在同一时刻，在多个核心上真正同时执行多个任务。

- 简言之：并发强调"同一时间段内交替处理"，并行强调"同一时刻同时执行"；并行是并发的一种特例，必须依赖多核硬件。

### 4.5 iOS中实现多线程的几种方案，各自有什么特点？

- NSThread 面向对象的，需要程序员手动创建线程，但不需要手动销毁。子线程间通信很难。

- GCD c语言，充分利用了设备的多核，自动管理线程生命周期。比NSOperation效率更高。

- NSOperation 基于gcd封装，更加面向对象，比gcd多了一些功能。

### 4.6 多个网络请求完成后执行下一步

- 使用GCD的dispatch_group_t

    创建一个dispatch_group_t

    每次网络请求前先dispatch_group_enter,请求回调后再dispatch_group_leave，enter和leave必须配合使用，有几次enter就要有几次leave，否则group会一直存在。

    当所有enter的block都leave后，会执行dispatch_group_notify的block。

    ```
    NSString *str = @"http://xxxx.com/";
    NSURL *url = [NSURL URLWithString:str];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSession *session = [NSURLSession sharedSession];

    dispatch_group_t downloadGroup = dispatch_group_create();
    for (int i=0; i<10; i++) {
        dispatch_group_enter(downloadGroup);

        NSURLSessionDataTask *task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
            NSLog(@"%d---%d",i,i);
            dispatch_group_leave(downloadGroup);
        }];
        [task resume];
    }

    dispatch_group_notify(downloadGroup, dispatch_get_main_queue(), ^{
        NSLog(@"end");
    });

    ```

- 使用GCD的信号量dispatch_semaphore_t

    dispatch_semaphore信号量为基于计数器的一种多线程同步机制。如果semaphore计数大于等于1，计数-1，返回，程序继续运行。如果计数为0，则等待。dispatch_semaphore_signal(semaphore)为计数+1操作,dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER)为设置等待时间，这里设置的等待时间是一直等待。

    思路：创建一个初值为 0 的信号量，每个请求完成后计数加一，当 10 个请求全部完成时发出信号；在子线程中等待该信号，等到后再继续后续逻辑。需要强调：此场景用 `dispatch_group` 更直接、更安全，信号量方案要点是对共享计数 `count` 做并发保护，且 `wait` 必须放在子线程，切勿阻塞主线程。

    ```
    NSString *str = @"http://xxxx.com/";
    NSURL *url = [NSURL URLWithString:str];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSession *session = [NSURLSession sharedSession];
    
    dispatch_semaphore_t sem = dispatch_semaphore_create(0);
    // completionHandler 可能在多条线程并发回调，故用串行队列保护计数，避免数据竞争
    dispatch_queue_t lockQueue = dispatch_queue_create("count.lock", DISPATCH_QUEUE_SERIAL);
    __block NSInteger count = 0;
    for (int i = 0; i < 10; i++) {
        NSURLSessionDataTask *task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
            dispatch_async(lockQueue, ^{
                count++;
                if (count == 10) {
                    dispatch_semaphore_signal(sem);
                }
            });
        }];
        [task resume];
    }
    // 务必在子线程等待，否则会阻塞主线程导致界面无响应
    dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"end");
    });
    
    ```

### 4.7 多个网络请求顺序执行后执行下一步

- 使用信号量semaphore

    每一次遍历，都让其dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER)，这个时候线程会等待，阻塞当前线程，直到dispatch_semaphore_signal(sem)调用之后

    ```
    NSString *str = @"http://www.jianshu.com/p/6930f335adba";
    NSURL *url = [NSURL URLWithString:str];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSession *session = [NSURLSession sharedSession];
    
    dispatch_semaphore_t sem = dispatch_semaphore_create(0);
    for (int i=0; i<10; i++) {
    
        NSURLSessionDataTask *task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
    
            NSLog(@"%d---%d",i,i);
            dispatch_semaphore_signal(sem);
        }];
    
        [task resume];
        dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    }
    
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"end");
    });
    
    ```


### 4.8 异步操作两组数据时, 执行完第一组之后, 才能执行第二组

- 这里使用 `dispatch_barrier_async` 栅栏方法即可实现。注意：栅栏函数只有作用于**自己创建的并发队列**（`DISPATCH_QUEUE_CONCURRENT`）时才有效；若传入全局并发队列（`dispatch_get_global_queue`），栅栏将不起作用。

    ```
    dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"第一次任务所在线程为: %@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"第二次任务所在线程为: %@", [NSThread currentThread]);
    });
    
    dispatch_barrier_async(queue, ^{
        NSLog(@"第一次任务, 第二次任务执行完毕, 继续执行");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"第三次任务所在线程为: %@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"第四次任务所在线程为: %@", [NSThread currentThread]);
    });
    
    ```

### 4.9 多线程中的死锁？

死锁是由于多个线程（进程）在执行过程中，因为争夺资源而造成的互相等待现象，你可以理解为卡住了。产生死锁的必要条件有四个：

- 互斥条件 ： 指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用毕释放。

- 请求和保持条件 ： 指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。

- 不可剥夺条件 ： 指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。

- 环路等待条件 ： 指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{P0，P1，P2，···，Pn}中的P0正在等待一个P1占用的资源；P1正在等待P2占用的资源，……，Pn正在等待已被P0占用的资源。

    最常见的就是"同步函数 + 主队列"的组合，本质是队列阻塞。以下代码需在主线程上执行：

    ```
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"2");
    });
    
    NSLog(@"1");
    // 该代码会造成死锁：当前主线程同步等待主队列执行 block，
    // 而主队列正被当前线程占用，二者互相等待，导致主线程永久阻塞（应用卡死）。
    
    ```

### 4.10 GCD执行原理？

- GCD 底层有一个线程池，池中存放着一个个线程。之所以称为“池”，是因为其中的线程可以被重用：当一个线程在一段时间内没有被调用时，就会被销毁。开多少条线程由底层线程池决定，无需开发者手动维护。开发者需要关心的，只是向队列中添加任务，再由队列负责调度。

- 如果向队列中提交的是同步任务，则任务会在当前调用线程上同步执行（不会开启新线程），执行完毕才返回。因此用 `currentThread` 打印时，得到的就是当前调用线程。

- 如果队列中存放的是异步的任务，（注意异步可以开线程），当任务出队后，底层线程池会提供一个线程供任务执行，因为是异步执行，队列中的任务不需等待当前任务执行完毕就可以调度下一个任务，这时底层线程池中会再次提供一个线程供第二个任务执行，执行完毕后再回到底层线程池中。

- 这样就实现了线程的复用，不必为每个任务都开启新线程，从而节约系统开销、提高效率。需要说明的是，能开启的线程数由系统根据当前负载等因素动态决定；实际开发中应避免无节制地创建线程，按需控制并发数量更为合理。


---

<a id="第五章block"></a>
## 第五章 Block


### 5.1 Objective-C 转 C++的方法

下面以 `TestClass.m` 为例说明 Block 转换后的结构，示例代码如下：

OC代码:

```
@interface TestClass ()
@end

@implementation TestClass
- (void)testMethods {
    void (^blockA)(int a) = ^(int a) {
        NSLog(@"%d",a);
    };
    if (blockA) {
        blockA(1990);
    }
}
@end
```

经过上述转换操作我们在TestClass.cpp中最下面发现如下代码

C++代码

```
// @interface TestClass ()
/* @end */

// @implementation TestClass

struct __TestClass__testMethods_block_impl_0 {
  struct __block_impl impl;
  struct __TestClass__testMethods_block_desc_0* Desc;
  __TestClass__testMethods_block_impl_0(void *fp, struct __TestClass__testMethods_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __TestClass__testMethods_block_func_0(struct __TestClass__testMethods_block_impl_0 *__cself, int a) {

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_wx_b8tcry0j24dbhr7zlzjq3v340000gn_T_TestClass_ee18d3_mi_0,a);
    }

static struct __TestClass__testMethods_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __TestClass__testMethods_block_desc_0_DATA = { 0, sizeof(struct __TestClass__testMethods_block_impl_0)};

static void _I_TestClass_testMethods(TestClass * self, SEL _cmd) {
    void (*blockA)(int a) = ((void (*)(int))&__TestClass__testMethods_block_impl_0((void *)__TestClass__testMethods_block_func_0, &__TestClass__testMethods_block_desc_0_DATA));
    if (blockA) {
        ((void (*)(__block_impl *, int))((__block_impl *)blockA)->FuncPtr)((__block_impl *)blockA, 1990);
    }
}
```

上面的代码生成是通过如下操作:

打开终端，cd到TestClass.m所在文件夹,使用如下命令
```
clang -rewrite-objc TestClass.m
```

就会在当前文件夹内自动生成对应的TestClass.cpp文件

> 注意：`clang` 随 Xcode 命令行工具一起提供。如果提示找不到 `clang`，执行下面的命令安装命令行工具即可：

```
xcode-select --install
# 安装完成后可用以下命令验证
clang --version
```

通过上述代码我们发现Block的其实是一个结构体类型

底层实现 会根据 `__`**类名**`__`**方法名**`_`block`_`impl`_`**下标** (0代表这个方法或者这个类中第0个block 下面如果还有将会 第1个block 第2个...)

```
struct __类名__方法名_block_impl_下标
```

### 5.2 关于变量的作用域

c语言的函数中可能使用的参数变量种类

- 参数类型
- 自动变量(局部变量)
- 静态变量(静态局部变量)
- 静态全局变量
- 全局变量

由于存储区域特殊,这其中有三种变量是可以在任何时候以任何状态调用的.

- 静态变量
- 静态全局变量
- 全局变量

而其他两种,则是有各自相应的作用域,超过作用域后,会被销毁.

---

### 5.3 block的内部实现，结构体是什么样的

看了上面的背景知识我们来回答一下这个问题

block的内部实现如下:

```
struct __TestClass__testMethods_block_impl_0 {
  struct __block_impl impl; //成员变量
  struct __TestClass__testMethods_block_desc_0* Desc; //desc 结构体声明
  // 构造函数
  // fp 函数指针
  // desc 静态全局变量初始化的 __main_block_desc_ 结构体实例指针
  // flags block 的负载信息(引用计数和类型信息),按位存储.
  __TestClass__testMethods_block_impl_0(void *fp, struct __TestClass__testMethods_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
//将来被调用的block内部的代码：block值被转换为C的函数代码
//这里，*__cself 是指向Block的值的指针，也就相当于是Block的值它自己(相当于C++里的this，
OC里的self)
//__cself 是指向__TestClass__testMethods_block_impl_0结构体实现的指针
//Block结构体就是__TestClass__testMethods_block_impl_0结构体.Block的值就是通过__TestClass__testMethods_block_impl_0构造出来的
static void __TestClass__testMethods_block_func_0(struct __TestClass__testMethods_block_impl_0 *__cself, int a) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_wx_b8tcry0j24dbhr7zlzjq3v340000gn_T_TestClass_9f58f7_mi_0,a);
}

static struct __TestClass__testMethods_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __TestClass__testMethods_block_desc_0_DATA = { 0, sizeof(struct __TestClass__testMethods_block_impl_0)};

static void _I_TestClass_testMethods(TestClass * self, SEL _cmd) {
    void (*blockA)(int a) = ((void (*)(int))&__TestClass__testMethods_block_impl_0((void *)__TestClass__testMethods_block_func_0, &__TestClass__testMethods_block_desc_0_DATA));
    if (blockA) {
        ((void (*)(__block_impl *, int))((__block_impl *)blockA)->FuncPtr)((__block_impl *)blockA, 1990);
    }
}
```

可以看得出来`__TestClass__testMethods_block_impl_0`有3个部分组成

- impl：类型为 `__block_impl` 的成员变量，内含 isa、Flags、Reserved 以及真正的函数指针 FuncPtr

```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;  //今后版本升级所需的区域
  void *FuncPtr; //函数指针
};
```

- Desc：指向 `__TestClass__testMethods_block_desc_0` 结构体的指针，用于描述该 block 的附加信息，包括 block 结构体的大小等

```
static struct __TestClass__testMethods_block_desc_0 {
  size_t reserved; //今后升级版本所需区域
  size_t Block_size; //block的大小
} __TestClass__testMethods_block_desc_0_DATA = { 0, sizeof(struct __TestClass__testMethods_block_impl_0)};

```

- `__TestClass__testMethods_block_impl_0()`构造函数,也就是该block的具体实现

```
__TestClass__testMethods_block_impl_0(void *fp, struct __TestClass__testMethods_block_desc_0 *desc, int flags=0) {
   impl.isa = &_NSConcreteStackBlock;
   impl.Flags = flags;
   impl.FuncPtr = fp;
   Desc = desc;
}
```

此结构体中

- isa 指针保存着所属类的指针。
- `struct __TestClass__testMethods_block_impl_0` 相当于 Block 的结构体（类比 Objective-C 的类对象结构体）。
- `_NSConcreteStackBlock` 相当于 Block 的类，isa 指向它；也就是说，**block 其实是 Objective-C 对闭包的对象化实现**。

讲到这里block的内部实现你看懂了吗?结构体是什么样的你记住了吗? 其实看着繁琐 细心观察代码会发现还是比较简单的.

### 5.4 block是类吗，有哪些类型?

block 本质上是一个对象，因为它带有 isa 指针。根据 isa 的指向，block 分为以下三种类型：

- `_NSConcreteGlobalBlock`：与全局变量类似，存放在程序的数据区域（.data）中。
- `_NSConcreteStackBlock`：位于栈上（前面讲的都是栈上的 block）。
- `_NSConcreteMallocBlock`：位于堆上。

> 这个isa可以按位运算

### 5.5 一个int变量被 `__block` 修饰与否的区别？block的变量截获

#### 5.5.1 被`__block` 修饰与否的区别

用一段示例代码来解答这个问题吧:

```
__block int a = 10;
int b = 20;

PrintTwoIntBlock block = ^(){
    a -= 10;
    printf("%d, %d\n",a,b);
};

block();//0 20

a += 20;
b += 30;

printf("%d, %d\n",a,b);//20 50

block();// 10 20
```

用 `__block` 修饰 `int a` 后，编译器会把 `a` 封装成一个 `__Block_byref_a_0` 结构体；block 截获的是指向该结构体的指针。这样 block 内部与外部访问的都是同一份存储，因此 block 内对 `a` 的修改能反映到外部，反之亦然。

`int b` 没有被 `__block` 修饰，block 仅按值截获 `b`（值拷贝）。因此在 block 内部无法修改外部的 `b`，外部对 `b` 的修改也不会影响 block 中已截获的值。

#### 5.5.2 block的变量截获

通过如下代码我们来观察要一下变量的捕获

```
blk_t blk;
{
    id array = [NSMutableArray new];
    blk = [^(id object){
        [array addObject:object];
        NSLog(@"array count = %ld",[array count]);
    } copy];
}
blk([NSObject new]);
blk([NSObject new]);
blk([NSObject new]);
```

输出打印

```
block_demo[28963:1629127] array count = 1
block_demo[28963:1629127] array count = 2
block_demo[28963:1629127] array count = 3
```

我们把上面的代码翻译成C++看下

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  id array;//截获的对象
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id _array, int flags=0) : array(_array) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

在Objc中，C结构体里不能含有被`__strong`修饰的变量，因为编译器不知道应该何时初始化和废弃C结构体。但是Objc的运行时库能够准确把握`Block`从栈复制到堆，以及堆上的block被废弃的时机，在实现上是通过`__TestClass__testMethods_block_copy_0`函数和`__TestClass__testMethods_block_dispose_0`函数进行的

```
static void __TestClass__testMethods_block_copy_0(struct __TestClass__testMethods_block_impl_0*dst, struct __TestClass__testMethods_block_impl_0*src) {
    _Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
static void __TestClass__testMethods_block_dispose_0(struct __TestClass__testMethods_block_impl_0*src) {
    _Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

- `_Block_object_assign`相当于retain操作,将对象赋值在对象类型的结构体成员变量中.

- `_Block_object_dispose`相当于release操作.

这两个函数调用的时机是在什么时候呢？

| 函数 | 被调用时机 |
| --- | --- |
| `__TestClass__testMethods_block_copy_0` | 从栈复制到堆时 |
| `__TestClass__testMethods_block_dispose_0` | 堆上的Block被废弃时 |

#### 5.5.3 什么时候栈上的Block会被复制到堆呢？

- 调用block的copy函数时。

- Block作为函数返回值返回时。

- 将Block赋值给附有`__strong`修饰符id类型的类或者Block类型成员变量时。

- 方法中含有usingBlock的Cocoa框架方法或者GCD的API中传递Block时。

#### 5.5.4 什么时候Block被废弃呢？

- 堆上的Block被释放后,谁都不再持有Block时调用dispose函数。

以上就是变量被block捕获的内容

---

### 5.6 `block`在修改`NSMutableArray`，需不需要添加`__block`

- 如修改`NSMutableArray`的存储内容的话,是不需要添加`__block`修饰的。
- 如修改`NSMutableArray`对象的本身,那必须添加`__block`修饰。

### 5.7 怎么进行内存管理的?

在上面Block的构造函数`__TestClass__testMethods_block_impl_0`中的isa指针指向的是&_NSConcreteStackBlock，它表示当前的Block位于栈区中.

| block内存操作 | 存储域/存储位置 | copy操作的影响 |
| --- | --- | --- |
| _NSConcreteGlobalBlock | 程序的数据区域 | 什么也不做 |
| _NSConcreteStackBlock | 栈 | 从栈拷贝到堆 |
| _NSConcreteMallocBlock | 堆 | 引用计数增加 |

- 全局Block:`_NSConcreteGlobalBlock`的结构体实例设置在程序的数据存储区，所以可以在程序的任意位置通过指针来访问，它的产生条件:
    - 记述全局变量的地方有block语法时.
    - block不截获的自动变量.

    > 以上两个条件只要满足一个就可以产生全局Block. [参考](https://juejin.im/post/6844903474312773646#heading-13)

- 栈Block:`_NSConcreteStackBlock`在生成Block以后，如果这个Block不是全局Block,那它就是栈Block,生命周期在其所属的变量作用域内.(也就是说如果销毁取决于所属的变量作用域).如果Block变量和`__block`变量复制到了堆上以后，则不再会受到变量作用域结束的影响了，因为它变成了堆Block.

- 堆Block:`_NSConcreteMallocBlock`将栈block复制到堆以后，block结构体的isa成员变量变成了`_NSConcreteMallocBlock`。

### 5.8 block可以用strong修饰吗?

在 ARC 中可以。当 block 被赋值给 `strong` 修饰的变量时，编译器会自动将其从栈复制到堆，因此用 `strong` 修饰能正确管理它的生命周期，无需开发者手动 copy。

在 MRC 中不行。`strong` 是 ARC 引入的关键字，MRC 并不支持；MRC 下必须用 `copy` 才能把栈 block 复制到堆，若改用 `retain`，则只会增加引用计数而不触发复制，block 仍留在栈上，作用域结束后即被销毁，再次调用就会崩溃。

### 5.9 解决循环引用时为什么要用`__strong`、`__weak`修饰?

首先，在 ARC 下，block 捕获 self（或访问其成员变量时隐式捕获 self）会对其产生强引用，从而形成默认的引用关系。因此通常在 block 外部先用 `__weak` 修饰目标对象，打破强引用环；而在 block 内部再用 `__strong` 临时持有这个弱引用。这样，当 block 从栈复制到堆、执行期间，对象会被 block 临时强持有，执行结束即释放——既避免了循环引用，又防止对象在 block 执行过程中被提前释放。若不这样处理，则很容易造成循环引用。

### 5.10 block发生copy时机?

在 ARC 中，编译器会在特定时机（如赋值给 `strong` 变量、作为函数返回值返回等）自动将栈 block 复制到堆；而 block 仅作为方法或函数的参数传递时，编译器不会执行 copy。常见的自动 copy 时机如下：

- 调用block的copy函数时。

- Block作为函数返回值返回时。

- 将Block赋值给附有`__strong`修饰符id类型的类或者Block类型成员变量时。

- 方法中含有usingBlock的Cocoa框架方法或者GCD的API中传递Block时。

### 5.11 Block访问对象类型的auto变量时，在ARC和MRC下有什么区别?

ARC下会对这个对象强引用，MRC下不会

[参考：Block 访问对象类型 auto 变量](https://juejin.im/post/6844903474312773646)


---

<a id="第六章通知机制nsnotification"></a>
## 第六章 通知机制（NSNotification）


### 6.1 通知中心的存储结构

首先通知中心结构大概分为如下几个类

- `NSNotification` 通知的模型 name、object、userinfo.
- `NSNotificationCenter`通知中心 负责发送`NSNotification`
- `NSNotificationQueue`通知队列 负责在某些时机触发 调用`NSNotificationCenter`通知中心 `post`通知

通知是结构体通过双向链表进行数据存储

```
// 根容器，NSNotificationCenter持有
typedef struct NCTbl {
  Observation		*wildcard;	/* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable		nameless;	/* 存储没有name但是有object的通知	*/
  GSIMapTable		named;		/* 存储带有name的通知，不管有没有object	*/
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct	Obs {
  id		observer;	/* 观察者，接收通知的对象	*/
  SEL		selector;	/* 响应方法		*/
  struct Obs	*next;		/* Next item in linked list.	*/
  ...
} Observation;
```

主要是以`key` `value`的形式存储,这里需要重点强调一下 通知以 `name`和`object`两个维度来存储相关通知内容,也就是我们添加通知的时候传入的两个不同的方法.

[![](https://upload-images.jianshu.io/upload_images/13277235-d1cdd2ef99a5c864.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable.jpg)
[![](https://upload-images.jianshu.io/upload_images/13277235-b25d70e69f5cb196.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable2.jpg)

简单理解`name`&`observer`&`SEL`之间的关系就是`name`作为`key`, `observer`作为观察者对象,当合适时机触发就会调用`observer`的`SEL`.

### 6.2 通知的发送是同步的，还是异步的

是同步发送。发送通知时，通知中心会在当前线程通过 `performSelector:` 逐一调用各观察者的响应方法，待所有观察者执行完毕后，`post` 方法才返回。而 `NSNotificationQueue` 所谓的"异步"，并非开启新线程，而是**非实时发送**、在**合适的时机发送**（借助 RunLoop 的时机触发）。

### 6.3 通知发送与接收线程

通知的接收线程与发送线程相同：在哪个线程调用 `post` 发送通知，观察者的响应方法就在哪个线程执行。因此若在子线程发送通知，响应方法也会在该子线程执行；如需在子线程异步发送，自行在子线程中调用 `post` 即可。

### 6.4 NSNotificationQueue是异步还是同步发送？在哪个线程响应

```
// 表示通知的发送时机
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送通知
    NSPostASAP = 2, // 尽快发送，这种时机是穿插在每次事件完成期间来做的
    NSPostNow = 3 // 立刻发送或者合并通知完成之后发送
};
```

|  | NSPostWhenIdle | NSPostASAP | NSPostNow |
| --- | --- | --- | --- |
| NSPostingStyle | 异步发送 | 异步发送 | 同步发送 |

`NSNotificationCenter`都是同步发送的，而这里介绍关于`NSNotificationQueue`的异步发送，从线程的角度看并不是真正的异步发送，或可称为**延时发送**，它是利用了`runloop`的时机来触发的.

异步线程发送通知则响应函数也是在异步线程,主线程发送则在主线程.

### 6.5 NSNotificationQueue和runloop的关系

`NSNotificationQueue`依赖`runloop`. 因为通知队列要在runloop回调的某个时机调用通知中心发送通知.从下面的枚举值就能看出来

```
// 表示通知的发送时机
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送通知
    NSPostASAP = 2, // 尽快发送，这种时机是穿插在每次事件完成期间来做的
    NSPostNow = 3 // 立刻发送或者合并通知完成之后发送
};
```

### 6.6 如何保证通知接收的线程在主线程

如果想在主线程响应异步通知的话可以用如下两种方式

1. 使用系统提供的注册接口指定回调队列

```
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block
```

2. `NSMachPort`的方式 通过在主线程的runloop中添加machPort，设置这个port的delegate，通过这个Port其他线程可以跟主线程通信，在这个port的代理回调中执行的代码肯定在主线程中运行，所以，在这里调用NSNotificationCenter发送通知即可

### 6.7 通知移除与页面销毁

iOS 9.0 之前，使用 `addObserver:selector:name:object:` 注册的观察者若在释放前未移除，会导致崩溃。原因是通知中心对观察者持有的是 `unsafe_unretained` 引用，观察者释放后其指针不会被置空，再次回调时即访问野指针。

iOS 9.0 之后，对于以 `addObserver:selector:name:object:` 注册的观察者，通知中心改为以 weak（zeroing weak）方式持有，观察者释放后指针自动置空，因此不再产生野指针崩溃，可不必手动移除。

但需特别注意：以 `addObserverForName:object:queue:usingBlock:` 注册的 **block 形式观察者并不受此保护**——通知中心会强持有它返回的 token，必须在合适时机手动调用 `removeObserver:` 移除，否则 block 仍可能被触发并访问已释放的对象。

### 6.8 多次添加同一个通知会是什么结果？多次移除通知呢

对同一组 name/object/selector 重复注册同一个观察者，会导致该通知发送一次时，响应方法被回调多次。重复移除观察者则不会产生崩溃。

### 6.9 通知 name 与 object 的匹配规则

```
// 注册观察者（接收通知），指定 object:@1
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
// 发送通知，object 传 nil
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
```

不能

首先我们看下通知中心存储通知观察者的结构

```
// 根容器，NSNotificationCenter持有
typedef struct NCTbl {
  Observation  *wildcard;    /* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable nameless;    /* 存储没有name但是有object的通知    */
  GSIMapTable named;        /* 存储带有name的通知，不管有没有object    */
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct Obs {
  id observer;    /* 观察者，接收通知的对象    */
  SEL selector;    /* 响应方法        */
  struct Obs *next;        /* Next item in linked list.    */
  ...
} Observation;
```

`nameless`与`named`的具体数据结构如下:

[![](https://upload-images.jianshu.io/upload_images/13277235-439098374929aec2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable.jpg)
[![](https://upload-images.jianshu.io/upload_images/13277235-03d732d55b6025d4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable2.jpg)

当添加通知监听的时候，我们传入了`name`和`object`，所以，观察者的存储链表是这样的：

`named`表：`key(name)` : `value`->`key(object)` : `value(Observation)`

因此发送通知时，若注册指定了非 nil 的 `object`，而发送时 `object` 为 nil，就无法在 `named` 表的二级（object → Observation）映射中命中对应节点，观察者回调不会执行。反之，注册时若 `object` 为 nil，则可接收该 `name` 下的所有通知，与发送方的 `object` 无关。


### 6.10 通知机制源码解析


本节以 GNUstep 源码为参考，梳理 `NSNotificationCenter` 的注册、存储、发送、删除和队列分发机制。

由于苹果没有开放相关源码，所以 GNUstep 不是官方实现，但对理解观察者模式的数据结构和分发流程有参考价值。

### 6.11 关键类结构


#### 6.11.1 NSNotification

用于描述通知的类，一个`NSNotification`对象就包含了一条通知的信息，所以当创建一个通知时通常包含如下属性：

```
@interface NSNotification : NSObject <NSCopying, NSCoding>
...
/* Querying a Notification Object */

- (NSString*) name; // 通知的name
- (id) object; // 携带的对象
- (NSDictionary*) userInfo; // 配置信息

@end
```

一般用于发送通知时使用，常用api如下：

```
- (void)postNotification:(NSNotification *)notification;
```

#### 6.11.2 NSNotificationCenter

这是个单例类，负责管理通知的创建和发送，属于最核心的类了。而`NSNotificationCenter`类主要负责三件事

1.  添加通知
2.  发送通知
3.  移除通知

核心`API`如下：

```
// 添加通知
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
// 发送通知
- (void)postNotification:(NSNotification *)notification;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;
// 删除通知
- (void)removeObserver:(id)observer;

```

#### 6.11.3 NSNotificationQueue

通知队列，用于异步发送消息，这个异步并不是开启线程，而是把通知存到双向链表实现的队列里面，等待某个时机触发时调用`NSNotificationCenter`的发送接口进行发送通知，这么看`NSNotificationQueue`最终还是调用`NSNotificationCenter`进行消息的分发

另外`NSNotificationQueue`是依赖`runloop`的，所以如果线程的`runloop`未开启则无效，至于为什么依赖`runloop`下面会解释

`NSNotificationQueue`主要做了两件事：

1.  添加通知到队列
2.  删除通知

核心`API`如下：

```
// 把通知添加到队列中，NSPostingStyle是个枚举，下面会介绍
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle;
// 删除通知，把满足合并条件的通知从队列中删除
- (void)dequeueNotificationsMatching:(NSNotification *)notification coalesceMask:(NSUInteger)coalesceMask;

```

#### 6.11.4 队列的合并策略和发送时机

把通知添加到队列等待发送，同时提供了一些附加条件供开发者选择，如：什么时候发送通知、如何合并通知等，系统给了如下定义

```
// 表示通知的发送时机
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送通知
    NSPostASAP = 2, // 尽快发送，这种情况稍微复杂，这种时机是穿插在每次事件完成期间来做的
    NSPostNow = 3 // 立刻发送或者合并通知完成之后发送
};
// 通知合并的策略，有些时候同名通知只想存在一个，这时候就可以用到它了
typedef NS_OPTIONS(NSUInteger, NSNotificationCoalescing) {
    NSNotificationNoCoalescing = 0, // 默认不合并
    NSNotificationCoalescingOnName = 1, // 只要name相同，就认为是相同通知
    NSNotificationCoalescingOnSender = 2  // object相同
};
```

#### 6.11.5 GSNotificationObserver

这个类是[GNUstep](https://github.com/gnustep/libs-base)源码中定义的，它的作用是代理观察者，主要用来实现接口：`addObserverForName：object: queue: usingBlock:`时用到，即要实现在指定队列回调block，那么`GSNotificationObserver`对象保存了`queue`和`block`信息，并且作为观察者注册到通知中心，等到接收通知时触发了响应方法，并在响应方法中把`block`抛到指定`queue`中执行，定义如下：

```
@implementation GSNotificationObserver
{
	NSOperationQueue *_queue; // 保存传入的队列
	GSNotificationBlock _block; // 保存传入的block
}
- (id) initWithQueue: (NSOperationQueue *)queue
               block: (GSNotificationBlock)block
{
......初始化操作
}

- (void) dealloc
{
....
}
// 响应接收通知的方法，并在指定队列中执行block
- (void) didReceiveNotification: (NSNotification *)notif
{
	if (_queue != nil)
	{
		GSNotificationBlockOperation *op = [[GSNotificationBlockOperation alloc]
			initWithNotification: notif block: _block];

		[_queue addOperation: op];
	}
	else
	{
		CALL_BLOCK(_block, notif);
	}
}

@end
```

### 6.12 存储容器

上面介绍了一些类的功能，但是要想实现通知中心的逻辑必须设计一套合理的存储结构，对于通知的存储基本上围绕下面几个结构体来做（大致了解下，后面章节会用到），后面会详细介绍具体逻辑的

```
// 根容器，NSNotificationCenter持有
typedef struct NCTbl {
  Observation		*wildcard;	/* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable		nameless;	/* 存储没有name但是有object的通知	*/
  GSIMapTable		named;		/* 存储带有name的通知，不管有没有object	*/
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct	Obs {
  id		observer;	/* 观察者，接收通知的对象	*/
  SEL		selector;	/* 响应方法		*/
  struct Obs	*next;		/* Next item in linked list.	*/
  ...
} Observation;

```

### 6.13 注册通知

本节以典型 API 为例分析通知注册流程，不同注册接口的核心存储逻辑一致。

本节聚焦 `NSNotificationCenter` 的注册流程，`NSNotificationQueue` 在后续小节单独说明。

#### 6.13.1 接口1

#### 6.13.2 注册源码

```
/*
observer：观察者，即通知的接收者
selector：接收到通知时的响应方法
name: 通知name
object：携带对象
*/
- (void) addObserver: (id)observer
	    	selector: (SEL)selector
                name: (NSString*)name
                object: (id)object {
  // 前置条件判断
  ......

  // 创建一个observation对象，持有观察者和SEL，下面进行的所有逻辑就是为了存储它
  o = obsNew(TABLE, selector, observer);

/*======= case1： 如果name存在 =======*/
  if (name) {
 	//-------- NAMED是个宏，表示名为named字典。以name为key，从named表中获取对应的mapTable
      n = GSIMapNodeForKey(NAMED, (GSIMapKey)(id)name);
      if (n == 0) { // 不存在，则创建
          m = mapNew(TABLE); // 先取缓存，如果缓存没有则新建一个map
          GSIMapAddPair(NAMED, (GSIMapKey)(id)name, (GSIMapVal)(void*)m);
          ...
	  }
      else { // 存在则把值取出来 赋值给m
          m = (GSIMapTable)n->value.ptr;
	  }
 	//-------- 以object为key，从字典m中取出对应的value，其实value被MapNode的结构包装了一层，这里不追究细节
      n = GSIMapNodeForSimpleKey(m, (GSIMapKey)object);
      if (n == 0) {// 不存在，则创建
          o->next = ENDOBS;
          GSIMapAddPair(m, (GSIMapKey)object, (GSIMapVal)o);
	  }
      else {
          list = (Observation*)n->value.ptr;
          o->next = list->next;
          list->next = o;
      }
    }
/*======= case2：如果name为空，但object不为空 =======*/
  else if (object) {
  	// 以object为key，从nameless字典中取出对应的value，value是个链表结构
      n = GSIMapNodeForSimpleKey(NAMELESS, (GSIMapKey)object);
      // 不存在则新建链表，并存到map中
      if (n == 0) {
          o->next = ENDOBS;
          GSIMapAddPair(NAMELESS, (GSIMapKey)object, (GSIMapVal)o);
	  }
      else { // 存在 则把值接到链表的节点上
		...
	  }
    }
/*======= case3：name 和 object 都为空 则存储到wildcard链表中 =======*/
  else {
      o->next = WILDCARD;
      WILDCARD = o;
  }
}
```

#### 6.13.3 逻辑说明

从上面介绍的`存储容器`中可以看出，`NCTable`结构体中核心的三个变量是：`wildcard`、`named`、`nameless`，在源码中分别用宏定义表示为：`WILDCARD`、`NAMELESS`、`NAMED`。

#### 6.13.4 case1: 存在`name`（无论object是否存在）

1.  注册通知，如果通知的`name`存在，则以`name`为key从`named`字典中取出值`n`(这个`n`其实被`MapNode`包装了一层，便于理解这里直接认为没有包装)，这个`n`还是个字典，各种判空新建逻辑不讨论
2.  然后以`object`为key，从字典`n`中取出对应的值，这个值就是`Observation`类型(后面简称`obs`)的链表，然后把刚开始创建的`obs`对象`o`存储进去

**数据结构关系图**

下面梳理通知观察者的存储关系：

![](https://upload-images.jianshu.io/upload_images/13277235-161480a9b966b17a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果注册通知时传入`name`，那么会是一个双层的存储结构

1.  找到`NCTable`中的`named`表，这个表存储了还有`name`的通知
2.  以`name`作为key，找到`value`，这个`value`依然是一个`map`
3.  `map`的结构是以`object`作为key，`obs`对象为value，这个`obs`对象的结构上面已经解释，主要存储了`observer & SEL`

#### 6.13.5 case2: 只存在object

1.  以`object`为key，从`nameless`字典中取出value，此value是个`obs`类型的链表
2.  把创建的`obs`类型的对象`o`存储到链表中

**数据结构关系图**

![](https://upload-images.jianshu.io/upload_images/13277235-6f91d1f3e88e5769.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只存在`object`时存储只有一层，那就是`object`和`obs`对象之间的映射

#### 6.13.6 case3: 没有name和object

这种情况直接把`obs`对象存放在了`Observation  *wildcard`  链表结构中

#### 6.13.7 接口2

#### 6.13.8 源码

**接口功能：** 此接口实现的功能是在接收到通知时，在指定队列`queue`执行`block`

```
// 这个api使用频率较低，怎么实现在指定队列回调block的，值得研究
- (id) addObserverForName: (NSString *)name
                   object: (id)object
                    queue: (NSOperationQueue *)queue
               usingBlock: (GSNotificationBlock)block
{
	// 创建一个临时观察者
	GSNotificationObserver *observer =
		[[GSNotificationObserver alloc] initWithQueue: queue block: block];
	// 调用了接口1的注册方法
	[self addObserver: observer
	         selector: @selector(didReceiveNotification:)
	             name: name
	           object: object];

	return observer;
}
```

#### 6.13.9 逻辑说明

这个接口依赖于`接口1`，只是多了一层代理观察者`GSNotificationObserver`，在`关键类结构`中已经介绍了它，设计思路值得学习

1.  创建一个`GSNotificationObserver`类型的对象`observer`，并把`queue`和`block`保存下来
2.  调用接口1进行通知的注册
3.  接收到通知时会响应`observer`的`didReceiveNotification:`方法，然后在`didReceiveNotification:`中把`block`抛给指定的`queue`去执行

#### 6.13.10 小结

1.  从上述介绍可以总结，存储是以`name`和`object`为维度的，即判定是不是同一个通知要从`name`和`object`区分，如果它们都相同则认为是同一个通知，后面包括查找逻辑、删除逻辑都是以这两个为维度的，这也解释了只传 `name` 不传 `object` 时为什么匹配不到观察者。
2.  理解数据结构的设计是整个通知机制的核心，其他功能只是在此基础上扩展了一些逻辑
3.  存储过程并没有做去重操作，这也解释了为什么同一个通知注册多次则响应多次

### 6.14 发送通知

#### 6.14.1 源码

发送通知的核心逻辑比较简单，基本上就是查找和调用响应方法，核心函数如下

```
// 发送通知
- (void) postNotificationName: (NSString*)name
		       object: (id)object
		     userInfo: (NSDictionary*)info
{
// 构造一个GSNotification对象， GSNotification继承了NSNotification
  GSNotification	*notification;
  notification = (id)NSAllocateObject(concrete, 0, NSDefaultMallocZone());
  notification->_name = [name copyWithZone: [self zone]];
  notification->_object = [object retain];
  notification->_info = [info retain];

  // 进行发送操作
  [self _postAndRelease: notification];
}
//发送通知的核心函数，主要做了三件事：查找通知、发送、释放资源
- (void) _postAndRelease: (NSNotification*)notification {
    //step1: 从named、nameless、wildcard表中查找对应的通知
    ...
    //step2：执行发送，即调用performSelector执行响应方法，从这里可以看出是同步的
   	[o->observer performSelector: o->selector
                    withObject: notification];
	//step3: 释放资源
    RELEASE(notification);
}

```

#### 6.14.2 逻辑说明

上述代码主要做了三件事：

1.  通过`name & object` 查找到所有的`obs`对象(保存了`observer`和`sel`)，放到数组中
2.  通过`performSelector：`逐一调用`sel`，这是个同步操作
3.  释放`notification`对象

#### 6.14.3 小结

从源码逻辑可以看出发送过程的概述：从三个存储容器中：`named`、`nameless`、`wildcard`去查找对应的`obs`对象，然后通过`performSelector：`逐一调用响应方法，这就完成了发送流程

**核心点：**

1.  同步发送
2.  遍历所有列表，即注册多次通知就会响应多次

### 6.15 删除通知

删除通知的源码主要是查找和删除逻辑，可以结合 [GNUstep 源码](https://github.com/gnustep/libs-base)理解。
**要注意的点：**

1.  查找时仍然以`name`和`object`为维度的，再加上`observer`做区分
2.  因为查找时做了这个链表的遍历，所以删除时会把重复的通知全都删除掉

```
// 删除已经注册的通知
- (void) removeObserver: (id)observer
		   name: (NSString*)name
                 object: (id)object {
  if (name == nil && object == nil && observer == nil)
      return;
      ...
}

- (void) removeObserver: (id)observer
{
  if (observer == nil)
    return;

  [self removeObserver: observer name: nil object: nil];
}
```

### 6.16 异步通知

`NSNotificationCenter` 是同步发送；`NSNotificationQueue` 的“异步”从线程角度看并不是真正开启异步线程，更准确地说是延时发送，它利用 `runloop` 的时机触发。

#### 6.16.1 入队

精简版源码如下，核心逻辑可以分为两步：

1.  根据`coalesceMask`参数判断是否合并通知
2.  接着根据`postingStyle`参数，判断通知发送的时机，如果不是立即发送则把通知加入到队列中：`_asapQueue`、`_idleQueue`

核心点：

1.  队列是双向链表实现
2.  当postingStyle值是立即发送时，调用的是`NSNotificationCenter`进行发送的，所以`NSNotificationQueue`还是依赖`NSNotificationCenter`进行发送

```
/*
* 把要发送的通知添加到队列，等待发送
* NSPostingStyle 和 coalesceMask在上面的类结构中有介绍
* modes这个就和runloop有关了，指的是runloop的mode
*/
- (void) enqueueNotification: (NSNotification*)notification
		postingStyle: (NSPostingStyle)postingStyle
		coalesceMask: (NSUInteger)coalesceMask
		    forModes: (NSArray*)modes
{
	......
  // 判断是否需要合并通知
  if (coalesceMask != NSNotificationNoCoalescing) {
      [self dequeueNotificationsMatching: notification
			    coalesceMask: coalesceMask];
  }
  switch (postingStyle) {
      case NSPostNow: {
      	...
      	// 如果是立马发送，则调用NSNotificationCenter进行发送
	     [_center postNotification: notification];
         break;
	  }
      case NSPostASAP:
      	// 添加到_asapQueue队列，等待发送
		add_to_queue(_asapQueue, notification, modes, _zone);
		break;

      case NSPostWhenIdle:
        // 添加到_idleQueue队列，等待发送
		add_to_queue(_idleQueue, notification, modes, _zone);
		break;
    }
}
```

#### 6.16.2 发送通知

发送通知的核心逻辑如下：

1.  `runloop`触发某个时机，调用`GSPrivateNotifyASAP()`和`GSPrivateNotifyIdle()`方法，这两个方法最终都调用了`notify()`方法
2.  `notify()`所做的事情就是调用`NSNotificationCenter`的`postNotification:`进行发送通知

```
static void notify(NSNotificationCenter *center,
                   NSNotificationQueueList *list,
                   NSString *mode, NSZone *zone)
{
 	......
    // 循环遍历发送通知
    for (pos = 0; pos < len; pos++)
	{
	  NSNotification	*n = (NSNotification*)ptr[pos];

	  [center postNotification: n];
	  RELEASE(n);
	}
	......
}
// 发送_asapQueue中的通知
void GSPrivateNotifyASAP(NSString *mode)
{
	notify(item->queue->_center,
	    item->queue->_asapQueue,
	    mode,
	    item->queue->_zone);
}
// 发送_idleQueue中的通知
void GSPrivateNotifyIdle(NSString *mode)
{
    notify(item->queue->_center,
    	item->queue->_idleQueue,
    	mode,
    	item->queue->_zone);
}

```

#### 6.16.3 小结

对于`NSNotificationQueue`总结如下

1.  依赖`runloop`，所以如果在其他子线程使用`NSNotificationQueue`，需要开启runloop
2.  最终还是通过`NSNotificationCenter`进行发送通知，所以这个角度讲它还是同步的
3.  所谓异步，指的是非实时发送而是在合适的时机发送，并没有开启异步线程

### 6.17 主线程响应通知

异步线程发送通知则响应函数也是在异步线程，如果执行UI刷新相关的话就会出问题，那么如何保证在主线程响应通知呢？

常见解决方式如下：

1.  使用`addObserverForName: object: queue: usingBlock`方法注册通知，指定在`mainqueue`上响应`block`
2.  在主线程注册一个 `machPort`，它用于线程通信。异步线程收到通知后向该 `machPort` 发送消息，主线程在 port 回调中处理通知。

---

<a id="第七章视图与图形"></a>
## 第七章 视图与图形


### 7.1 AutoLayout的原理，性能如何?

#### 7.1.1 AutoLayout的原理

> 来历 一般大家都会认为Auto Layout这个东西是苹果自己搞出来的，其实不然，早在1997年Alan Borning, Kim Marriott, Peter Stuckey等人就发布了《Solving Linear Arithmetic Constraints for User Interface Applications》论文（[论文地址:http://constraints.cs.washington.edu/solvers/uist97.html](http://constraints.cs.washington.edu/solvers/uist97.html)）提出了在解决布局问题的Cassowary constraint-solving算法实现，并且将代码发布在他们搭建的[Cassowary网站上http://constraints.cs.washington.edu/cassowary/](http://constraints.cs.washington.edu/cassowary/)。后来更多开发者用各种语言来写Cassowary，比如说pybee用python写的https://github.com/pybee/cassowary。自从它发布以来JavaScript，.NET，JAVA，Smalltalk 和C++都有相应的库。2011年苹果将这个算法运用到了自家的布局引擎中，美其名曰Auto Layout。

**AutoLayout的原理就是用Cassowary算法来将布局问题抽象成线性不等式，并分解成多个位置间的约束**
因为多了计算视图大小frame的过程,所以性能肯定没有指定Frame坐标要快.

参考：[深入剖析Auto Layout，分析iOS各版本新增特性](http://www.starming.com/2015/11/03/deeply-analyse-autolayout/)

#### 7.1.2 性能如何?

下面是[WWDC2018 High Performance Auto Layout](https://developer.apple.com/videos/play/wwdc2018/220/)中对比的iOS12和iOS11下分别使用自动布局的性能对比现场.

[![](https://upload-images.jianshu.io/upload_images/13277235-7f3b7a19e8e4e67b.gif?imageMogr2/auto-orient/strip)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/HighPerformanceAutoLayoutiOS11iOS12Compare.gif)

经过实验得出如下图标结论:

[![](https://upload-images.jianshu.io/upload_images/13277235-b9325441a3364189.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/HighPerformanceAutoLayoutResult.png)

iOS12之前，视图嵌套的数量对性能的影响是呈指数级增长的，而iOS12优化之后对性能的影响是线性增长，对性能消耗不大。

无论如何优化也肯定不如CGRectFrame那样的设置更加直接,性能更好.

### 7.2 UIView & CALayer的区别

| 区别 | UIView | CALayer |
| --- | --- | --- |
| 继承父类 | UIView:UIResponder:NSObject | CALayer:NSObject |
| 用途 | 可以处理触摸事件 | 不处理用户的交互,不参与响应事件传递 |
| 两者关系 | 有一个CALayer成员变量 eg: view.layer | 是UIView的成员变量 |
| 分工 | 处理交互层事件并包装各种图形的简单设置 | 底层渲染图形,支持动画 |

### 7.3 事件响应链

参考：[iOS触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

### 7.4 drawrect & layoutsubviews调用时机

`layoutSubviews:`(相当于layoutSubviews()函数)在以下情况下会被调用：

1.  init初始化不会触发layoutSubviews。
2.  addSubview会触发layoutSubviews。
3.  设置view的Frame会触发layoutSubviews (frame发生变化触发)。
4.  滚动一个UIScrollView会触发layoutSubviews。
5.  旋转Screen会触发父UIView上的layoutSubviews事件。
6.  改变一个UIView大小的时候也会触发父UIView上的layoutSubviews事件。
7.  调用 `setNeedsLayout` 会在下一轮 RunLoop 触发 `layoutSubviews`；调用 `layoutIfNeeded` 则会立即触发布局（注意：并不存在 `setLayoutSubviews` 这个方法）。

`drawRect:` 在以下情况下会被调用：

1.  视图首次显示、需要绘制内容时被调用，通常发生在 `loadView`、`viewDidLoad` 之后的渲染阶段。
2.  改变 UIView 的 `contentMode` 或 `frame`（尺寸发生变化）时，可能触发系统调用 `drawRect:`。
3.  调用 `setNeedsDisplay` 或 `setNeedsDisplayInRect:` 标记后，系统会在下一个渲染周期回调 `drawRect:`。

> 知识点扩充: 当我们操作drawRect方法的时候实际是在操作内存中存放视图的backingStore区域,用于后续图形的渲染操作,如果不理解可以看下[UIView的渲染过程](https://www.jianshu.com/p/a120d6c64d88).

### 7.5 UI的刷新原理

UI 刷新可以从 RunLoop 和渲染流水线两个角度理解。

iOS 屏幕通常以 60Hz 刷新（约每 16.67ms 一帧）。主线程在每一帧的渲染时机（由 VSync 信号经 CoreAnimation / CADisplayLink 驱动）需要完成以下工作：

- view 的缓冲区创建；
- view 内容的绘制（如果重写了 drawRect）；
- 接收并处理系统的触摸事件。

我们看到的 UI 图形，实际上是 CPU 和 GPU 不断配合工作的结果。

正是由于主线程的 RunLoop 持续运转，UI 才获得了刷新与事件处理的时机——无论是界面渲染还是触摸事件响应，都依赖 RunLoop 的不断驱动。主线程的 RunLoop 默认处于启动状态，以便持续响应用户交互。

### 7.6 隐式动画 & 显式动画区别

- 隐式动画：直接修改单独 CALayer 的可动画属性（如 position、opacity 等）时，Core Animation 会自动生成的过渡动画。它默认存在，可通过事务（CATransaction）或重写图层的 action 来关闭。

- 显式动画：开发者手动创建 CAAnimation（如 CABasicAnimation、CAKeyframeAnimation 等）并通过 `addAnimation:forKey:` 添加到图层上的动画。

需要注意：UIView 默认关闭了其根 layer 的隐式动画（layer action 返回 nil），因此直接修改 UIView.layer 的属性不会产生隐式动画；隐式动画通常以独立创建的 CALayer 为例来演示。

### 7.7 imageNamed 与 imageWithContentsOfFile 区别

| 区别 | imageNamed: | imageWithContentsOfFile: |
| --- | --- | --- |
| 是否缓存 | 会将解码后的图片缓存到系统管理的内存缓存中 | 不缓存，每次都重新加载 |
| 适用场景 | 频繁使用的小图（缓存可能在收到内存警告时才释放） | 大图或仅使用一次的图片 |

### 7.8 什么是离屏渲染

离屏渲染（Off-Screen Rendering）指 GPU 在当前屏幕缓冲区（frame buffer）之外，另外开辟一块离屏缓冲区进行渲染，渲染完成后再合成回屏幕的过程。由于需要额外创建缓冲区、并在屏幕缓冲区与离屏缓冲区之间切换上下文，会带来额外的性能开销。

常见触发离屏渲染的场景：

- `cornerRadius` 配合 `masksToBounds` 对内容做圆角裁剪；
- 设置 `layer.mask` 遮罩；
- 设置阴影（未指定 `shadowPath` 时）；
- 开启光栅化 `shouldRasterize = YES`；
- 组透明 `allowsGroupOpacity`、抗锯齿等。

[![image](https://upload-images.jianshu.io/upload_images/13277235-acd41dc1f938b9bc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/CoreAnimationPipeline.jpg)

参考：[iOS 离屏渲染的深入研究](https://zhuanlan.zhihu.com/p/72653360)

### 7.9 多个相同的图片，会重复加载吗

取决于加载方式：通过 `imageNamed:` 加载时，系统会缓存解码结果，相同名称的图片不会重复解码，可直接复用缓存；而 `imageWithContentsOfFile:` 不走缓存，每次调用都会重新读取并解码，因此会重复加载。

### 7.10 图片是什么时候解码的，如何优化

图片在首次需要显示前才会被解码。流程大致为：图片数据经 `CGImageSourceCreateWithData` 创建 `CGImageSource`，再生成 `CGImage`，最终解码为可供渲染的 bitmap 位图。该解码默认由 Core Animation 在图层提交（commit）、送入 GPU 渲染流水线之前自动完成，解码后的位图存放在图层的 backingStore 中。

#### 7.10.1 如何优化

核心思路是：在子线程提前对图片进行强制解码（例如将其绘制到一个 bitmap context 中得到已解码的位图），避免主线程在渲染时同步解码而造成卡顿。

可借助以 `CGImageSource` 为前缀的相关 API（如 `CGImageSourceCreateThumbnailAtIndex`），在合适的时机手动控制图片的解码与缩放，并合理利用系统资源，从而构建一套占用内存小、加载快的图片加载缓存库。

可参考开源库 [PINRemoteImage](https://github.com/pinterest/PINRemoteImage) 或 [YYWebImage](https://github.com/ibireme/YYWebImage)。

### 7.11 图片渲染怎么优化

可从以下方面入手：为阴影显式指定 `shadowPath`；用预先裁切好的圆角图或贝塞尔路径替代 `cornerRadius + masksToBounds`；在子线程提前解码图片；避免离屏渲染；按需降低图片分辨率以减少内存与采样开销，从而提升帧率、降低功耗。

[iOS 开发-视图渲染与性能优化](https://www.jianshu.com/p/748f9abafff8)

### 7.12 如果GPU的刷新率超过了iOS屏幕60Hz刷新率是什么现象，怎么解决

现象是**画面撕裂（tearing）**：当 GPU 的出帧速率超过屏幕刷新率（60Hz）时，屏幕在一次刷新周期内可能读取到分属两帧的画面数据，导致上下部分来自不同帧，出现撕裂。

解决方案是开启**垂直同步（VSync）**，并配合**双缓冲 / 三重缓冲**机制：只在 VSync 信号到来时交换缓冲区，使屏幕每次都显示完整的一帧，从而消除撕裂。iOS 系统本身已采用 VSync + 双缓冲机制。此外，可借助 Instruments 的 Core Animation、GPU 等工具检测渲染瓶颈并定位优化点。

---

<a id="第八章数据结构"></a>
## 第八章 数据结构


### 8.1 数据结构的存储方式

数据结构的存储一般常用的有两种 顺序存储结构 和 链式存储结构

- 顺序存储结构:

    比如，数组，1-2-3-4-5-6-7-8-9-10，存储是按顺序的。再比如栈和队列等

- 链式存储结构:

    以链表为例：逻辑上仍是 1→2→3→…→10 的有序序列，但各结点在内存中的物理位置可以任意分散，靠每个结点保存的后继结点地址（指针）串联起来。也就是说，逻辑顺序不变，变的是物理存储位置不再连续。

### 8.2 集合结构 线性结构 树形结构 图形结构

- 集合结构

    一个集合，就是一个圆圈中有很多个元素，元素与元素之间没有任何关系 这个很简单

- 线性结构

    可以想象成一条线上站着很多个人，这条线不一定是直的，也可以是弯的或分段的。线性结构中元素之间是一对一的关系

- 树形结构

    做开发的肯定或多或少的知道xml 解析 树形结构跟他非常类似。也可以想象成一个金字塔。树形结构是一对多的关系

- 图形结构

    图结构相对复杂，顶点之间是多对多的关系（可分为有向图与无向图），类似于人际关系网中的相互交集

### 8.3 单向链表 双向链表 循环链表

- 单向链表 A->B->C->D->E->F->G->H. 这就是单向链表，A 是头结点，H 是尾结点，像一列只有一个车头的火车一样，由车头（A 端）牵引向前 ![单向链表](https://upload-images.jianshu.io/upload_images/17495317-be8ca79b70959032.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 双向链表 ![双向链表](https://upload-images.jianshu.io/upload_images/17495317-11ef8cd9bd858e7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 循环链表

    循环链表是与单向链表一样，是一种链式的存储结构，所不同的是，循环链表的最后一个结点的指针是指向该循环链表的第一个结点或者表头结点，从而构成一个环形的链。发挥想象力 A->B->C->D->E->F->G->H->A. 绕成一个圈。就像蛇吃自己的这就是循环 不需要去死记硬背哪些理论知识。

### 8.4 数组和链表区别

- 数组

    数组元素在内存上连续存放，可以通过下标查找元素；插入、删除需要移动大量元素，比较适用于元素很少变化的情况

- 链表

    链表中的元素在内存中不是顺序存储的，查找慢，插入、删除只需要对元素指针重新赋值，效率高

### 8.5 堆、栈和队列

#### 8.5.1 堆

- 堆是一种经过排序的树形数据结构，每个节点都有一个值，通常我们所说的堆的数据结构是指二叉树。所以堆在数据结构中通常可以被看做是一棵树的数组对象。而且堆需要满足一下两个性质：

    1）堆中某个节点的值总是不大于或不小于其父节点的值；

    2）堆总是一棵完全二叉树。

- 堆分为两种情况，有最大堆和最小堆。将根节点最大的堆叫做最大堆或大根堆，根节点最小的堆叫做最小堆或小根堆，在一个摆放好元素的最小堆中，父结点中的元素一定比子结点的元素要小，但对于左右结点的大小则没有规定谁大谁小。

- 堆常用来实现优先队列，堆的存取是随意的，这就如同我们在图书馆的书架上取书，虽然书的摆放是有顺序的，但是我们想取任意一本时不必像栈一样，先取出前面所有的书，书架这种机制不同于箱子，我们可以直接取出我们想要的书。

#### 8.5.2 栈

- 栈是限定仅在表尾进行插入和删除操作的线性表。我们把允许插入和删除的一端称为栈顶，另一端称为栈底，不含任何数据元素的栈称为空栈。栈的特殊之处在于它限制了这个线性表的插入和删除位置，它始终只在栈顶进行。

- 栈是一种具有后进先出的数据结构，又称为后进先出的线性表，简称 LIFO（Last In First Out）结构。也就是说后存放的先取，先存放的后取，这就类似于我们要在取放在箱子底部的东西（放进去比较早的物体），我们首先要移开压在它上面的物体（放进去比较晚的物体）。

- 堆栈中定义了一些操作。两个最重要的是PUSH和POP。PUSH操作在堆栈的顶部加入一个元素。POP操作相反，在堆栈顶部移去一个元素，并将堆栈的大小减一。

- 栈的应用—递归

#### 8.5.3 队列

- 队列是只允许在一端进行插入操作、而在另一端进行删除操作的线性表。允许插入的一端称为队尾，允许删除的一端称为队头。它是一种特殊的线性表，特殊之处在于它只允许在表的前端进行删除操作，而在表的后端进行插入操作，和栈一样，队列是一种操作受限制的线性表。

- 队列是一种先进先出的数据结构，又称为先进先出的线性表，简称 FIFO（First In First Out）结构。也就是说先放的先取，后放的后取，就如同行李过安检的时候，先放进去的行李在另一端总是先出来，后放入的行李会在最后面出来。

### 8.6 输入一棵二叉树的根结点，求该树的深度？

二叉树的结点定义如下：

```
struct BinaryTreeNode
{
	int m_nValue;
	BinaryTreeNode* m_pLeft;
	BinaryTreeNode* m_pRight;
};

```

- 如果一棵树只有一个结点，它的深度为1。
- 如果根结点只有左子树而没有右子树，那么树的深度应该是其左子树的深度加1；同样如果根结点只有右子树而没有左子树，那么树的深度应该是其右子树的深度加1。
- 如果既有右子树又有左子树，那该树的深度就是其左、右子树深度的较大值再加1。

```
int TreeDepth(BinaryTreeNode* pRoot)
{
    if(pRoot == nullptr)
        return 0;
    int left = TreeDepth(pRoot->m_pLeft);
    int right = TreeDepth(pRoot->m_pRight);

    return (left > right) ? (left + 1) : (right + 1);
}

```

### 8.7 输入一棵二叉树的根结点，判断该树是不是平衡二叉树？

- 重复遍历结点

    先求出根结点的左右子树的深度，然后判断它们的深度相差是否不超过 1，如果否，则不是平衡二叉树；如果是，再用同样的方法分别判断左子树和右子树是否为平衡二叉树，如果都是，则这就是一棵平衡二叉树。

- 遍历一遍结点

    遍历结点的同时记录下该结点的深度，避免重复访问。

方法1:

```
struct TreeNode{
    int val;
    TreeNode* left;
    TreeNode* right;
};

int TreeDepth(TreeNode* pRoot){
    if(pRoot==NULL)
        return 0;
    int left=TreeDepth(pRoot->left);
    int right=TreeDepth(pRoot->right);
    return left>right?(left+1):(right+1);
}

bool IsBalanced(TreeNode* pRoot){
    if(pRoot==NULL)
        return true;
    int left=TreeDepth(pRoot->left);
    int right=TreeDepth(pRoot->right);
    int diff=left-right;
    if(diff>1 || diff<-1)
        return false;
    return IsBalanced(pRoot->left) && IsBalanced(pRoot->right);
}

```

方法2：

```
bool IsBalanced_1(TreeNode* pRoot,int& depth){
    if(pRoot==NULL){
        depth=0;
        return true;
    }
    int left,right;
    int diff;
    if(IsBalanced_1(pRoot->left,left) && IsBalanced_1(pRoot->right,right)){
        diff=left-right;
        if(diff>=-1 && diff<=1){   // 左右子树高度差的绝对值不超过 1 才平衡
            depth=left>right?left+1:right+1;
            return true;
        }
    }
    return false;
}

bool IsBalancedTree(TreeNode* pRoot){
    int depth=0;
    return IsBalanced_1(pRoot,depth);
}
```


---

<a id="第九章算法"></a>
## 第九章 算法


### 9.1 时间复杂度

- 时间频度

    一个算法执行所耗费的时间,从理论上是不能算出来的,必须上机运行测试才能知道.但我们不可能也没有必要对每个算法都上机测试,只需知道哪个算法花费的时间多,哪个算法花费的时间少就可以了.并且一个算法花费的时间与算法中语句的执行次数成正比例,哪个算法中语句执行次数多,它花费时间就多.一个算法中的语句执行次数称为语句频度或时间频度.记为T(n).

- 时间复杂度

    一般情况下,算法中基本操作重复执行的次数是问题规模n的某个函数,用T(n)表示,若有某个辅助函数f(n),使得当n趋近于无穷大时,T（n)/f(n)的极限值为不等于零的常数,则称f(n)是T(n)的同数量级函数.记作T(n)=O(f(n)),称O(f(n)) 为算法的渐进时间复杂度,简称时间复杂度.

- 在各种不同算法中,若算法中语句执行次数为一个常数,则时间复杂度为O(1),另外,在时间频度不相同时,时间复杂度有可能相同,如T(n)=n2+3n+4与T(n)=4n2+2n+1它们的频度不同,但时间复杂度相同,都为O(n2).

- 按数量级递增排列,常见的时间复杂度有：

    O(1)称为常量级，算法的时间复杂度是一个常数。

    O(n)称为线性级，时间复杂度是数据量n的线性函数。

    O(n²)称为平方级，与数据量n的二次多项式函数属于同一数量级。

    O(n³)称为立方级，是n的三次多项式函数。

    O(logn)称为对数级，是n的对数函数。

    O(nlogn)称为介于线性级和平方级之间的一种数量级

    O(2ⁿ)称为指数级，与数据量n的指数函数是一个数量级。

    O(n!)称为阶乘级，与数据量n的阶乘是一个数量级。

    它们之间的关系是： O(1)<O(logn)<O(n)<O(nlogn)<O(n²)<O(n³)<O(2ⁿ)<O(n!)，随着问题规模n的不断增大,上述时间复杂度不断增大,算法的执行效率越低.

### 9.2 空间复杂度

- 评估执行程序所需的存储空间。可以估算出程序对计算机内存的使用程度。不包括算法程序代码和所处理的数据本身所占空间部分。通常用所使用额外空间的字节数表示。其算法比较简单，记为S(n)=O(f(n))，其中，n表示问题规模。

### 9.3 常用的排序算法

- 选择排序、冒泡排序、插入排序三种排序算法可以总结为如下：

    都将数组分为已排序部分和未排序部分。

    选择排序将已排序部分定义在左端，然后选择未排序部分的最小元素和未排序部分的第一个元素交换。

    冒泡排序将已排序部分定义在右端，在遍历未排序部分的过程执行交换，将最大元素交换到最右端。

    插入排序将已排序部分定义在左端，将未排序部分元的第一个元素插入到已排序部分合适的位置。

```
/**
 *	【选择排序】：最值出现在起始端
 *
 *	第1趟：在n个数中找到最小(大)数与第一个数交换位置
 *	第2趟：在剩下n-1个数中找到最小(大)数与第二个数交换位置
 *	重复这样的操作...依次与第三个、第四个...数交换位置
 *	第n-1趟，最终可实现数据的升序（降序）排列。
 *
 */
void selectSort(int *arr, int length) {
    for (int i = 0; i < length - 1; i++) { //趟数
        for (int j = i + 1; j < length; j++) { //比较次数
            if (arr[i] > arr[j]) {
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
    }
}

/**
 *	【冒泡排序】：相邻元素两两比较，比较完一趟，最值出现在末尾
 *	第1趟：依次比较相邻的两个数，不断交换（小数放前，大数放后）逐个推进，最值最后出现在第n个元素位置
 *	第2趟：依次比较相邻的两个数，不断交换（小数放前，大数放后）逐个推进，最值最后出现在第n-1个元素位置
 *	 ……   ……
 *	第n-1趟：依次比较相邻的两个数，不断交换（小数放前，大数放后）逐个推进，最值最后出现在第2个元素位置
 */
void bubbleSort(int *arr, int length) {
    for(int i = 0; i < length - 1; i++) { //趟数
        for(int j = 0; j < length - i - 1; j++) { //比较次数
            if(arr[j] > arr[j+1]) {
                int temp = arr[j];
                arr[j] = arr[j+1];
                arr[j+1] = temp;
            }
        }
    }
}

/**
 *	折半查找：优化查找时间（不用遍历全部数据）
 *
 *	折半查找的原理：
 *   1> 数组必须是有序的
 *   2> 必须已知min和max（知道范围）
 *   3> 动态计算mid的值，取出mid对应的值进行比较
 *   4> 如果mid对应的值大于要查找的值，那么max要变小为mid-1
 *   5> 如果mid对应的值小于要查找的值，那么min要变大为mid+1
 *
 */

// 已知一个有序数组, 和一个key, 要求从数组中找到key对应的索引位置
int findKey(int *arr, int length, int key) {
    int min = 0, max = length - 1, mid;
    while (min <= max) {
        mid = (min + max) / 2; //计算中间值
        if (key > arr[mid]) {
            min = mid + 1;
        } else if (key < arr[mid]) {
            max = mid - 1;
        } else {
            return mid;
        }
    }
    return -1;
}

```

### 9.4 字符串反转

```
void char_reverse (char *cha) {

    // 定义头部指针
    char *begin = cha;
    // 定义尾部指针
    char *end = cha + strlen(cha) -1;

    while (begin < end) {

        char temp = *begin;
        *(begin++) = *end;
        *(end--) = temp;
    }
}

```

### 9.5 链表反转（头插法）

.h声明文件

```
#import <Foundation/Foundation.h>

// 定义一个链表
struct Node {
    int data;
    struct Node *next;
};

@interface ReverseList : NSObject

// 链表反转
struct Node* reverseList(struct Node *head);

// 构造一个链表
struct Node* constructList(void);

// 打印链表中的数据
void printList(struct Node *head);

@end

```

.m实现文件

```
#import "ReverseList.h"

@implementation ReverseList

struct Node* reverseList(struct Node *head)
{
    // 定义遍历指针，初始化为头结点
    struct Node *p = head;

    // 反转后的链表头部
    struct Node *newH = NULL;

    // 遍历链表
    while (p != NULL) {

        // 记录下一个结点
        struct Node *temp = p->next;
        // 当前结点的next指向新链表头部
        p->next = newH;
        // 更改新链表头部为当前结点
        newH = p;
        // 移动p指针
        p = temp;
    }

    // 返回反转后的链表头结点
    return newH;
}

struct Node* constructList(void)
{
    // 头结点定义
    struct Node *head = NULL;
    // 记录当前尾结点
    struct Node *cur = NULL;

    for (int i = 1; i < 5; i++) {
        struct Node *node = malloc(sizeof(struct Node));
        node->data = i;

        // 头结点为空，新结点即为头结点
        if (head == NULL) {
            head = node;
        }
        // 当前结点的next为新结点
        else{
            cur->next = node;
        }

        // 设置当前结点为新结点
        cur = node;
    }

    return head;
}

void printList(struct Node *head)
{
    struct Node* temp = head;
    while (temp != NULL) {
        printf("node is %d \n", temp->data);
        temp = temp->next;
    }
}

@end

```

### 9.6 有序数组合并

.h声明文件

```
#import <Foundation/Foundation.h>

@interface MergeSortedList : NSObject
// 将有序数组a和b的值合并到一个数组result当中，且仍然保持有序
void mergeList(int a[], int aLen, int b[], int bLen, int result[]);

@end

```

.m实现文件

```
#import "MergeSortedList.h"

@implementation MergeSortedList

void mergeList(int a[], int aLen, int b[], int bLen, int result[])
{
    int p = 0; // 遍历数组a的指针
    int q = 0; // 遍历数组b的指针
    int i = 0; // 记录当前存储位置

    // 任一数组没有到达边界则进行遍历
    while (p < aLen && q < bLen) {
        // 如果a数组对应位置的值小于b数组对应位置的值
        if (a[p] <= b[q]) {
            // 存储a数组的值
            result[i] = a[p];
            // 移动a数组的遍历指针
            p++;
        }
        else{
            // 存储b数组的值
            result[i] = b[q];
            // 移动b数组的遍历指针
            q++;
        }
        // 指向合并结果的下一个存储位置
        i++;
    }

    // 如果a数组有剩余
    while (p < aLen) {
        // 将a数组剩余部分拼接到合并结果的后面
        result[i] = a[p++];
        i++;
    }

    // 如果b数组有剩余
    while (q < bLen) {
        // 将b数组剩余部分拼接到合并结果的后面
        result[i] = b[q++];
        i++;
    }
}

@end

```

### 9.7 查找第一个只出现一次的字符（Hash查找）

.h声明文件

```
#import <Foundation/Foundation.h>

@interface HashFind : NSObject

// 查找第一个只出现一次的字符
char findFirstChar(char* cha);

@end

```

.m实现文件

```
#import "HashFind.h"

@implementation HashFind

char findFirstChar(char* cha)
{
    char result = '\0';

    // 定义一个数组 用来存储各个字母出现次数
    int array[256];

    // 对数组进行初始化操作
    for (int i=0; i<256; i++) {
        array[i] =0;
    }
    // 定义一个指针 指向当前字符串头部
    char* p = cha;
    // 遍历每个字符
    while (*p != '\0') {
        // 在字母对应存储位置 进行出现次数+1操作
        array[*(p++)]++;
    }

    // 将P指针重新指向字符串头部
    p = cha;
    // 遍历每个字母的出现次数
    while (*p != '\0') {
        // 遇到第一个出现次数为1的字符，打印结果
        if (array[*p] == 1)
        {
            result = *p;
            break;
        }
        // 反之继续向后遍历
        p++;
    }

    return result;
}

@end

```

### 9.8 查找两个子视图的共同父视图

.h声明文件

```
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>
@interface CommonSuperFind : NSObject

// 查找两个视图的共同父视图
- (NSArray<UIView *> *)findCommonSuperView:(UIView *)view other:(UIView *)viewOther;

@end

```

.m实现文件

```
#import "CommonSuperFind.h"

@implementation CommonSuperFind

- (NSArray <UIView *> *)findCommonSuperView:(UIView *)viewOne other:(UIView *)viewOther
{
    NSMutableArray *result = [NSMutableArray array];

    // 查找第一个视图的所有父视图
    NSArray *arrayOne = [self findSuperViews:viewOne];
    // 查找第二个视图的所有父视图
    NSArray *arrayOther = [self findSuperViews:viewOther];

    int i = 0;
    // 越界限制条件
    while (i < MIN((int)arrayOne.count, (int)arrayOther.count)) {
        // 倒序方式获取各个视图的父视图
        UIView *superOne = [arrayOne objectAtIndex:arrayOne.count - i - 1];
        UIView *superOther = [arrayOther objectAtIndex:arrayOther.count - i - 1];

        // 比较如果相等 则为共同父视图
        if (superOne == superOther) {
            [result addObject:superOne];
            i++;
        }
        // 如果不相等，则结束遍历
        else{
            break;
        }
    }

    return result;
}

- (NSArray <UIView *> *)findSuperViews:(UIView *)view
{
    // 初始化为第一父视图
    UIView *temp = view.superview;
    // 保存结果的数组
    NSMutableArray *result = [NSMutableArray array];
    while (temp) {
        [result addObject:temp];
        // 顺着superview指针一直向上查找
        temp = temp.superview;
    }
    return result;
}

@end

```

### 9.9 无序数组中的中位数(快排思想)

.h声明文件

```
#import <Foundation/Foundation.h>

@interface MedianFind : NSObject

// 无序数组中位数查找
int findMedian(int a[], int aLen);

@end

```

```
.m实现文件
#import "MedianFind.h"

@implementation MedianFind

//求一个无序数组的中位数
int findMedian(int a[], int aLen)
{
    int low = 0;
    int high = aLen - 1;

    int mid = (aLen - 1) / 2;
    int div = PartSort(a, low, high);

    while (div != mid)
    {
        if (mid < div)
        {
            //左半区间找
            div = PartSort(a, low, div - 1);
        }
        else
        {
            //右半区间找
            div = PartSort(a, div + 1, high);
        }
    }
    //找到了
    return a[mid];
}

int PartSort(int a[], int start, int end)
{
    int low = start;
    int high = end;

    //选取关键字
    int key = a[end];

    while (low < high)
    {
        //左边找比key大的值
        while (low < high && a[low] <= key)
        {
            ++low;
        }

        //右边找比key小的值
        while (low < high && a[high] >= key)
        {
            --high;
        }

        if (low < high)
        {
            //找到之后交换左右的值
            int temp = a[low];
            a[low] = a[high];
            a[high] = temp;
        }
    }

    int temp = a[high];
    a[high] = a[end];
    a[end] = temp;

    return low;
}

@end

```


### 9.10 给定一个整数数组和一个目标值，找出数组中和为目标值的两个数。

```
- (void)viewDidLoad {

    [super viewDidLoad];

    NSArray *oriArray = @[@(2),@(3),@(6),@(7),@(22),@(12)];

    BOOL isHaveNums =  [self twoNumSumWithTarget:9 Array:oriArray];

    NSLog(@"%d",isHaveNums);
}

- (BOOL)twoNumSumWithTarget:(int)target Array:(NSArray<NSNumber *> *)array {

    NSMutableArray *finalArray = [NSMutableArray array];

    for (int i = 0; i < array.count; i++) {

        for (int j = i + 1; j < array.count; j++) {

            if ([array[i] intValue] + [array[j] intValue] == target) {

                [finalArray addObject:array[i]];
                [finalArray addObject:array[j]];
                NSLog(@"%@",finalArray);

                return YES;
            }
        }
    }
    return NO;
}
```


---

<a id="第十章性能优化"></a>
## 第十章 性能优化


### 10.1 造成tableView卡顿的原因有哪些？

- 1.最常用的就是cell的重用， 注册重用标识符

    如果不重用cell时，每当一个cell显示到屏幕上时，就会重新创建一个新的cell

    如果有很多数据的时候，就会堆积很多cell。

    如果重用cell，为cell创建一个ID，每当需要显示cell 的时候，都会先去缓冲池中寻找可循环利用的cell，如果没有再重新创建cell

- 2.避免cell的重新布局

    cell的布局填充等操作 比较耗时，一般创建时就布局好

    如可以将cell单独放到一个自定义类，初始化时就布局好

- 3.提前计算并缓存cell的属性及内容

    在 UITableView 的数据源回调中，系统并不是先创建 cell 再确定其高度，而是先确定每个 cell 的高度，高度确定后才创建要显示的 cell。

    滚动时，cell 每次进入屏幕都会触发高度计算。通过提前估算高度（estimatedRowHeight），系统先按预估值确定 contentSize，再在 cell 即将显示时调用具体的高度计算方法，从而避免为屏幕外的 cell 浪费计算时间。

- 4.减少cell中控件的数量

    尽量使 cell 的布局大致相同；不同风格的 cell 应使用不同的重用标识符；初始化时一次性添加好控件，

    暂时用不到的先隐藏

- 5.不要使用ClearColor，无背景色，透明度也不要设置为0

    渲染耗时比较长

- 6.使用局部更新

    如果只需更新某一分区，使用 reloadSections: 进行局部刷新

- 7.加载网络数据，下载图片，使用异步加载，并缓存

- 8.尽量减少在 cellForRow 中通过 addSubview 给 cell 动态添加子视图

- 9.按需加载cell，cell滚动很快时，只加载范围内的cell

- 10.不要实现无用的代理方法，tableView只遵守两个协议

- 11.缓存行高：estimatedHeightForRow不能和HeightForRow里面的layoutIfNeed同时存在，这两者同时存在才会出现“窜动”的bug。所以我的建议是：只要是固定行高就写预估行高来减少行高调用次数提升性能。如果是动态行高就不要写预估方法了，用一个行高的缓存字典来减少代码的调用次数即可

- 12.不要做多余的绘制工作。在实现drawRect:的时候，它的rect参数就是需要绘制的区域，这个区域之外的不需要进行绘制。例如上例中，就可以用CGRectIntersectsRect、CGRectIntersection或CGRectContainsRect判断是否需要绘制image和text，然后再调用绘制方法。

- 13.预渲染图像。当新的图像出现时，仍然会有短暂的停顿现象。解决的办法就是在bitmap context里先将其画一遍，导出成UIImage对象，然后再绘制到屏幕；

- 14.使用正确的数据结构来存储数据。

### 10.2 如何提升 tableview 的流畅度？

- 本质上是降低 CPU、GPU 的工作，从这两个大的方面去提升性能。

    CPU：对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制

    GPU：纹理的渲染

- 卡顿优化在 CPU 层面

    尽量用轻量级的对象，比如用不到事件处理的地方，可以考虑使用 CALayer 取代 UIView

    不要频繁地调用 UIView 的相关属性，比如 frame、bounds、transform 等属性，尽量减少不必要的修改

    尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性

    Autolayout 会比直接设置 frame 消耗更多的 CPU 资源

    图片的 size 最好刚好跟 UIImageView 的 size 保持一致

    控制一下线程的最大并发数量

    尽量把耗时的操作放到子线程

    文本处理（尺寸计算、绘制）

    图片处理（解码、绘制）

- 卡顿优化在 GPU层面

    尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示

    GPU能处理的最大纹理尺寸是 4096x4096，一旦超过这个尺寸，就会占用 CPU 资源进行处理，所以纹理尽量不要超过这个尺寸

    尽量减少视图数量和层次

    减少透明的视图（alpha<1），不透明的就设置 opaque 为 YES

    尽量避免出现离屏渲染

- iOS 保持界面流畅的技巧

    1. 预排版，提前计算

    在接收到服务端返回的数据后，尽量将 CoreText 排版的结果、单个控件的高度、cell 整体的高度提前计算好，将其存储在模型的属性中。需要使用时，直接从模型中往外取，避免了计算的过程。

    尽量少用 UILabel，可以使用 CALayer 。避免使用 AutoLayout 的自动布局技术，采取纯代码的方式

    2. 预渲染，提前绘制

    例如圆形的图标可以提前在，在接收到网络返回数据时，在后台线程进行处理，直接存储在模型数据里，回到主线程后直接调用就可以了

    避免使用 CALayer 的 Border、corner、shadow、mask 等技术，这些都会触发离屏渲染。

    3. 异步绘制

    4. 全局并发线程

    5. 高效的图片异步加载

### 10.3 APP启动时间应从哪些方面优化？

App启动时间可以通过xcode提供的工具来度量，在Xcode的Product->Scheme-->Edit Scheme->Run->Arguments中，将环境变量DYLD_PRINT_STATISTICS设为YES，优化需以下方面入手

- dylib loading time

    核心思想是减少dylibs的引用

    合并现有的dylibs（最好是6个以内）

    使用静态库

- rebase/binding time

    核心思想是减少DATA块内的指针

    减少 Objective-C 元数据量：精简类、分类的数量，减少不必要的实例变量与方法（需与面向对象设计权衡）

    减少c++虚函数

    多使用Swift结构体（推荐使用swift）

- ObjC setup time

    核心思想同上，这部分内容基本上在上一阶段优化过后就不会太过耗时

    initializer time

- 尽量避免重写 `+load` 方法（它在 pre-main 阶段同步执行，会拖慢启动），将初始化逻辑延后到 `+initialize` 或真正使用时再执行

    减少使用c/c++的attribute((constructor))；推荐使用dispatch_once() pthread_once() std:once()等方法

    推荐使用swift

    不要在初始化中调用dlopen()方法，因为加载过程是单线程，无锁，如果调用dlopen则会变成多线程，会开启锁的消耗，同时有可能死锁

    不要在初始化中创建线程

### 10.4 如何降低APP包的大小

降低包大小需要从两方面着手

- 可执行文件

    编译器优化：Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default 设置为 YES，去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions 设置为 NO， Other C Flags 添加 -fno-exceptions 利用 AppCode 检测未使用的代码：菜单栏 -> Code -> Inspect Code

    编写LLVM插件检测出重复代码、未被调用的代码

- 资源（图片、音频、视频 等）

    优化的方式可以对资源进行无损的压缩

    去除没有用到的资源

### 10.5 如何检测离屏渲染与优化

- 检测：在模拟器菜单 Debug -> Color Off-screen Rendered，或使用 Instruments 的 Core Animation 工具勾选 Color Offscreen-Rendered Yellow，被离屏渲染的区域会显示为黄色。

- 优化，如阴影，在绘制时添加阴影的路径

### 10.6 怎么检测图层混合

1. 模拟器debug中color blended layers红色区域表示图层发生了混合

2. Instrument-选中Core Animation-勾选Color Blended Layers

避免图层混合：

- 确保控件的opaque属性设置为true，确保backgroundColor和父视图颜色一致且不透明

- 如无特殊需要，不要设置低于1的alpha值

- 确保UIImage没有alpha通道

UILabel图层混合解决方法：

iOS 8 及以后，将背景色设为不透明并设置 `label.layer.masksToBounds = YES`，使 label 仅渲染其实际 size 区域，即可解决 UILabel 的图层混合问题

iOS8 之前只要设置背景色为非透明的就行

为什么设置了背景色但是在iOS8上仍然出现了图层混合呢？

UILabel在iOS8前后的变化，在iOS8以前，UILabel使用的是CALayer作为底图层，而在iOS8开始，UILabel的底图层变成了_UILabelLayer，绘制文本也有所改变。在背景色的四周多了一圈透明的边，而这一圈透明的边明显超出了图层的矩形区域，设置图层的masksToBounds为YES时，图层将会沿着Bounds进行裁剪 图层混合问题解决了

### 10.7 日常如何检查内存泄露？

- 目前我知道的方式有以下几种

    Memory Leaks

    Allocations

    Analyse

    Debug Memory Graph

    MLeaksFinder

- 泄露的内存主要有以下两种：

    Leaked Memory 这种是忘记 Release 操作所泄露的内存。

    Abandoned Memory 这种是循环引用，无法释放掉的内存。


---

<a id="第十一章组件化"></a>
## 第十一章 组件化


### 11.1 组件化有什么好处？

- 业务分层、解耦，使代码变得可维护；

- 有效的拆分、组织日益庞大的工程代码，使工程目录变得可维护；

- 便于各业务功能拆分、抽离，实现真正的功能复用；

- 业务隔离，跨团队开发代码控制和版本风险控制的实现；

- 模块化对代码的封装性、合理性都有一定的要求，提升开发同学的设计能力；

- 在维护好各级组件的情况下，随意组合满足不同客户需求；（只需要将之前的多个业务组件模块在新的主App中进行组装即可快速迭代出下一个全新App）

### 11.2 你是如何组件化解耦的？

- 分层

    基础功能组件：按功能分库，不涉及产品业务需求，跟库Library类似，通过良好的接口拱上层业务组件调用；不写入产品定制逻辑，通过扩展接口完成定制；

    基础UI组件：各个业务模块依赖使用，但需要保持好定制扩展的设计

    业务组件：业务功能间相对独立，相互间没有Model共享的依赖；业务之间的页面调用只能通过UIBus进行跳转；业务之间的逻辑Action调用只能通过服务提供；

- 中间件：target-action，url-block，protocol-class

### 11.3 为什么CTMediator方案优于基于Router的方案？

Router的缺点：

- 在组件化的实施过程中，注册URL并不是充分必要条件。组件是不需要向组件管理器注册URL的，注册了URL之后，会造成不必要的内存常驻。注册URL的目的其实是一个服务发现的过程，在iOS领域中，服务发现的方式是不需要通过主动注册的，使用runtime就可以了。另外，注册部分的代码的维护是一个相对麻烦的事情，每一次支持新调用时，都要去维护一次注册列表。如果有调用被弃用了，是经常会忘记删项目的。runtime由于不存在注册过程，那就也不会产生维护的操作，维护成本就降低了。 由于通过runtime做到了服务的自动发现，拓展调用接口的任务就仅在于各自的模块，任何一次新接口添加，新业务添加，都不必去主工程做操作，十分透明。

- 在iOS领域里，一定是组件化的中间件为openURL提供服务，而不是openURL方式为组件化提供服务。如果在给App实施组件化方案的过程中是基于openURL的方案的话，有一个致命缺陷：非常规对象(不能被字符串化到URL中的对象，例如UIImage)无法参与本地组件间调度。 在本地调用中使用URL的方式其实是不必要的，如果业务工程师在本地间调度时需要给出URL，那么就不可避免要提供params，在调用时要提供哪些params是业务工程师很容易懵逼的地方。

- 为了支持传递非常规参数，蘑菇街的方案采用了protocol，这个会侵入业务。由于业务中的某个对象需要被调用，因此必须要符合某个可被调用的protocol，然而这个protocol又不存在于当前业务领域，于是当前业务就不得不依赖public Protocol。这对于将来的业务迁移是有非常大的影响的。

CTMediator的优点：

- 调用时，区分了本地应用调用和远程应用调用。本地应用调用为远程应用调用提供服务。

- 组件仅通过Action暴露可调用接口，模块与模块之间的接口被固化在了Target-Action这一层，避免了实施组件化的改造过程中，对Business的侵入，同时也提高了组件化接口的可维护性。

- 方便传递各种类型的参数。

### 11.4 基于CTMediator的组件化方案，有哪些核心组成？

- CTMediator中间件：集成就可以了

- 模块Target_%@：模块的实现及提供对外的方法调用Action_methodName，需要传参数时，都统一以NSDictionary*的形式传入。

- CTMediator+%@扩展：扩展里声明了模块业务的对外接口，参数明确，这样外部调用者可以很容易理解如何调用接口。


---

<a id="第十二章综合面试题与扩展专题"></a>
## 第十二章 综合面试题与扩展专题


### 12.1 iOS内存管理机制

iOS内存管理机制的原理是引用计数，当这块内存被创建后，它的引用计数0->1，表示有一个对象或指针持有这块内存，拥有这块内存的所有权，如果这时候有另外一个对象或指针指向这块内存，那么为了表示这个后来的对象或指针对这块内存的所有权，引用计数1->2，之后若有一个对象或指针不再指向这块内存时，引用计数-1，表示这个对象或指针不再拥有这块内存的所有权，当一块内存的引用计数变为0，表示没有任何对象或指针持有这块内存，系统便会立刻释放掉这块内存。

- alloc、new ：类初始化方法，开辟新的内存空间，引用计数+1；
- retain ：实例方法，不会开辟新的内存空间，引用计数+1；
- copy : 实例方法，对源对象做拷贝。需注意"浅拷贝/深拷贝"与"copy/mutableCopy"是两个不同维度的概念：浅拷贝只复制指针、不开辟新内存（引用计数+1），深拷贝复制内容、会开辟新内存。对不可变对象 copy 是浅拷贝（返回原对象本身）；对可变对象 copy 或 mutableCopy 才会分配新内存。
- strong ：强引用，引用计数+1；
- release ：实例方法，释放对象，引用计数-1；
- autorelease : 延迟释放，加入 autoreleasepool 自动释放池，在池子销毁时引用计数-1；
- assign : 直接赋值，不改变引用计数。它既能作用于对象，也能作用于基本数据类型；所指向的对象销毁时不会自动置 nil，可能产生野指针。
- weak : 弱引用，不改变引用计数，只能作用于对象；所指向的对象销毁时会自动置为 nil，从而避免野指针。

### 12.2 NSThread、GCD、NSOperation多线程

1. NSThread

> NSThread是封装程度最小最轻量级的，使用更灵活，但要手动管理线程的生命周期、线程同步和线程加锁等，开销较大；

```
[NSThread isMultiThreaded];//BOOL 是否开启了多线程
[NSThread currentThread];//NSThread 获取当前线程
[NSThread mainThread];//NSThread 获取主线程
[NSThread sleepForTimeInterval:1];//线程睡眠1s

```

2. GCD

> GCD基于C语言封装的，遵循FIFO

```
dispatch_sync与dispatch_async//同步和异步操作

dispatch_queue_t;//主要有串行和并发两种；
	其中：
	dispatch_queue_create("concurrent_queue", DISPATCH_QUEUE_CONCURRENT)并发；
	dispatch_queue_create("serial_queue", DISPATCH_QUEUE_SERIAL)串行；

dispatch_once_t;//代码只会被执行一次,用于单例
dispatch_after；//延迟操作
dispatch_get_main_queue;//回到主线程操作

//Demo单例
+ (instancetype)sharedInstance {
    static ZZScreenshotsMonitor *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

//Demo：执行顺序
- (void)viewDidLoad {
    [super viewDidLoad];
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"1");
    });

    NSLog(@"2");

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND,0);

    dispatch_sync(queue, ^{
        NSLog(@"3");
    });

    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"4");
    });

    dispatch_async(queue, ^{
        NSLog(@"5");
    });

    NSLog(@"6");

    [self performSelector:@selector(delayMethod) withObject:nil afterDelay:0];

    NSLog(@"8");
}

- (void)delayMethod {
	NSLog(@"7");
}

打印结果：23658147；其中5和8随机调换

```

- NSOperation

> NSOperation基于GCD封装的，比GCD可控性更强;可以加入操作依赖（addDependency）、设置操作队列最大可并发执行的操作个数（setMaxConcurrentOperationCount）、取消操作（cancel）等,需要使用两个它的实体子类：NSBlockOperation和NSInvocationOperation，或者继承NSOperation自定义子类;NSBlockOperation 和 NSInvocationOperation 用法的主要区别是：NSBlockOperation 执行 Block（代码块），NSInvocationOperation 执行指定的方法（target/selector），相对来说前者（NSBlockOperation）更加灵活易用。NSOperation操作配置完成后便可调用start函数在当前线程执行，如果要异步执行避免阻塞当前线程则可以加入NSOperationQueue中异步执行

### 12.3 输入一个字符串，判断这个字符串是否是有效的IP地址

```
+ (BOOL)isValidIP:(NSString *)ipStr {
    if (nil == ipStr) {
        return NO;
    }

    NSArray *ipArray = [ipStr componentsSeparatedByString:@"."];
    if (ipArray.count == 4) {
        for (NSString *ipnumberStr in ipArray) {
			  if ([self isPureInt:ipnumberStr]) {
				  int ipnumber = [ipnumberStr intValue];
	            if (!(ipnumber>=0 && ipnumber<=255)) {
	                return NO;
	            }
			  }
        }
        return YES;
    }
    return NO;
}
//是否整形
- (BOOL)isPureInt:(NSString*)string {
	NSScanner* scan = [NSScanner scannerWithString:string];
	int val;
	return[scan scanInt:&val] && [scan isAtEnd];
}
//是否只含有数字
- (BOOL)validateNumber:(NSString*)number {
	BOOL res = YES;
	NSCharacterSet* tmpSet = [NSCharacterSet characterSetWithCharactersInString:@"0123456789"];
	int i = 0;
	while (i < number.length) {
	    NSString * string = [number substringWithRange:NSMakeRange(i, 1)];
	    NSRange range = [string rangeOfCharacterFromSet:tmpSet];
	    if (range.length == 0) {
	        res = NO;
	        break;
	    }
	    i++;
	}
	return res;

}

```

### 12.4 大数加法怎么实现？

> 使用字符串实现；

```
/两个大数相加算法
-(NSString *)addTwoNumberWithOneNumStr:(NSString *)one anotherNumStr:(NSString *)another
{
    int i = 0;
    int j = 0;
    int maxLength = 0;
    int sum = 0;
    int overflow = 0;
    int carryBit = 0;
    NSString *temp1 = @"";
    NSString *temp2 = @"";
    NSString *sums = @"";
    NSString *tempSum = @"";
    int length1 = (int)one.length;
    int length2 = (int)another.length;
    //1.反转字符串
    for (i = length1 - 1; i >= 0 ; i--) {
        NSRange range = NSMakeRange(i, 1);
        temp1 = [temp1 stringByAppendingString:[one substringWithRange:range]];
        NSLog(@"%@",temp1);
    }
    for (j = length2 - 1; j >= 0; j--) {
        NSRange range = NSMakeRange(j, 1);
        temp2 = [temp2 stringByAppendingString:[another substringWithRange:range]];
        NSLog(@"%@",temp2);
    }

    //2.补全缺少位数为0
    maxLength = length1 > length2 ? length1 : length2;
    if (maxLength == length1) {
        for (i = length2; i < length1; i++) {
            temp2 = [temp2 stringByAppendingString:@"0"];
            NSLog(@"i = %d --%@",i,temp2);
        }
    }else{
        for (j = length1; j < length2; j++) {
            temp1 = [temp1 stringByAppendingString:@"0"];
            NSLog(@"j = %d --%@",j,temp1);
        }
    }
    //3.取数做加法
    for (i = 0; i < maxLength; i++) {
        NSRange range = NSMakeRange(i, 1);
        int a = [temp1 substringWithRange:range].intValue;
        int b = [temp2 substringWithRange:range].intValue;
        sum = a + b + carryBit;
        if (sum > 9) {
            if (i == maxLength -1) {
                overflow = 1;
            }
            carryBit = 1;
            sum -= 10;
        }else{
            carryBit = 0;
        }
        tempSum = [tempSum stringByAppendingString:[NSString stringWithFormat:@"%d",sum]];
    }
    if (overflow == 1) {
        tempSum = [tempSum stringByAppendingString:@"1"];
    }
    int sumlength = (int)tempSum.length;
    for (i = sumlength - 1; i >= 0 ; i--) {
        NSRange range = NSMakeRange(i, 1);
        sums = [sums stringByAppendingString:[tempSum substringWithRange:range]];
    }
    NSLog(@"sums = %@",sums);
    return sums;
}

```

### 12.5 简述KVC和KVO，其中KVO实现原理？

> KVC : 键值编码（Key-Value Coding），它是一种通过 key 访问对象属性的机制，而不必直接调用 setter/getter 方法。以 `setValue:forKey:` 为例，其底层查找机制如下：
> 1. 首先按 `setKey:`、`_setKey:` 的顺序查找对应的 setter 方法，找到则调用并赋值。
> 2. 如果没有找到 setter 方法，则检查 `+ (BOOL)accessInstanceVariablesDirectly` 是否返回 YES（默认返回 YES）。若返回 YES，则按 `_key`、`_isKey`、`key`、`isKey` 的顺序查找成员变量，找到则直接对其赋值。
> 3. 如果上述都未找到（或 `accessInstanceVariablesDirectly` 返回 NO），则调用兜底方法 `- (void)setValue:(id)value forUndefinedKey:(NSString *)key`，其默认实现会抛出 `NSUnknownKeyException` 异常。

> KVO : 键值观察者 （Key-Value Observer）: KVO 是观察者模式的一种实现，观察者A监听被观察者B的某个属性，当B的属性发生更改时，A就会收到通知，执行相应的方法。实现原理：基本的原理：当观察某对象A时，KVO机制动态创建一个对象A当前类的子类，并为这个新的子类重写了被观察属性keyPath的setter 方法。setter 方法随后负责通知观察对象属性的改变状况。

> Apple 使用了 isa 混写（isa-swizzling）来实现 KVO 。当观察对象A时，KVO机制动态创建一个新的名为：NSKVONotifying_A 的新类，该类继承自对象A的本类，且 KVO 为 NSKVONotifying_A 重写观察属性的 setter 方法，setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象属性值的更改情况。

> 子类setter方法剖析：KVO 的键值观察通知依赖于 NSObject 的两个方法:willChangeValueForKey:和 didChangeValueForKey:，在存取数值的前后分别调用 2 个方法：
> 被观察属性发生改变之前，willChangeValueForKey:被调用，通知系统该 keyPath 的属性值即将变更；当改变发生后， didChangeValueForKey: 被调用，通知系统该 keyPath 的属性值已经变更；之后， observeValueForKeyPath:ofObject:change:context: 也会被调用。且重写观察属性的 setter 方法这种继承方式的注入是在运行时而不是编译时实现的。

> 如果赋值没有通过 setter 方法或者 KVC，而是直接修改属性对应的成员变量，例如：仅调用 _name = @“newName”，这时是不会触发 KVO 机制，更加不会调用回调方法的

### 12.6 Block实现原理；堆上和栈上的数据如何同步？

> block本质上也是一个oc对象，他内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。结构体，在栈上的情况, Block中的指针只是指向栈上的__block变量, 而当Block/__block变量被copy到堆上以后, 堆上Block会持有堆上__block变量. 而堆上的Block再次被调用copy时, 只是Block的引用计数+1而已, 而__block变量如果被多个堆上Block持有也只涉及到引用记数的变化. 一旦Block/__block变量的引用计数为0, 就会自动从堆上释放内存.这里Block/__block变量在堆上的内存管理与Objective-C对象完全一致.

```
Block类					原存储域	调用copy效果
_NSConcreteStackBlock	栈			从栈copy到堆
_NSConcreteGlobalBlock	数据域(.data域)	什么也不做
_NSConcreteMallocBlock	堆			引用计数+1

```

### 12.7 iOS设计模式

- 构造模式

> 构造模式用于将某个业务的属性和行为进行分离，当业务属性越多的时候该模式用起来就越方便。比如：我要自定义一个比较灵活的弹窗，这个弹窗有显示和隐藏、动画的功能，并且弹窗的大小、样式显示的位置都要可以自定义。这样我们就可以使用构造模式，将行为和属性分离出来，弹窗的显示和隐藏就是行为，其他的均为属性，这些属性的构造过程中就可以被定义好.

- 适配器模式；

> 1：何为适配器模式？
> 适配器模式将一个类的接口适配成用户所期待的。一个适配器通常允许因为接口不兼容而不能一起工作的类能够在一起工作，做法是将类自己的接口包裹在一个已存在的类中。
> 2：[如何使用适配器模式？]
> 当你想使用一个已经存在的类，而它的接口不符合你的需求；
> 你想创建一个可以复用的类，该类可以与其他不相关的类或不可预见的类协同工作；
> 你想使用一些已经存在的子类，但是不可能对每一个都进行子类化以匹配它们的接口，对象适配器可以适配它的父亲接口。
> 3：[适配器模式的优缺点？]
> 优点：降低数据层和视图层（对象）的耦合度，使之使用更加广泛，适应复杂多变的变化。
> 缺点：降低了可读性，代码量增加，对于不理解这种模式的人来说比较难看懂。

- 策略模式;

> 1:何为策略模式？策略模式定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变化。
> 2:如何使用策略模式？
> 在有多种算法相似的情况下，使用 if…else 所带来的复杂和难以维护。
> 如果在一个系统里面有许多类，它们之间的区别仅在于它们的行为，那么使用策略模式可以动态地让一个对象在许多行为中选择一种行为。
> 一个系统需要动态地在几种算法中选择一种。
> 如果一个对象有很多的行为，如果不用恰当的模式，这些行为就只好使用多重的条件选择语句来实现。
> 注意事项：如果一个系统的策略多于四个，就需要考虑使用混合模式，解决策略类膨胀的问题。
> 3:策略模式的优缺点？
> 优点：简化操作，提高代码维护性。算法可以自由切换，避免使用多重条件判断，扩展性良好。
> 缺点：在使用之前就要确定使用某种策略，而不是动态的选择策略。策略类会增多，所有策略类都需要对外暴露。

- 观察者模式;

> 1:[何为观察者模式？]
> 当对象间存在一对多关系时，则使用观察者模式（Observer Pattern）。比如，当一个对象被修改时，则会自动通知它的依赖对象。观察者模式属于行为型模式。
> 2:如何使用观察者模式？
> 一个对象状态改变给其他对象通知的问题，而且要考虑到易用和低耦合，保证高度的协作。一个对象（目标对象）的状态发生改变，所有的依赖对象（观察者对象）都将得到通知，进行广播通知。
> 3:观察者模式的优缺点？
> 优点：观察者和被观察者是抽象耦合的。建立一套触发机制。缺点：如果一个被观察者对象有很多的直接和间接的观察者的话，将所有的观察者都通知到会花费很多时间。如果在观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃。观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化。

- 原型/外观模式;

> 1:何为原型/外观模式？
> 原型模式：（Prototype Pattern）用于创建重复的对象，同时又能保证性能。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。这种模式是实现了一个原型接口，该接口用于创建当前对象的克隆。当直接创建对象的代价比较大时，则采用这种模式。
> 外观模式：（Facade Pattern）隐藏系统的复杂性，并向客户端提供了一个客户端可以访问系统的接口。这种类型的设计模式属于结构型模式，它向现有的系统添加一个接口，来隐藏系统的复杂性。这种模式涉及到一个单一的类，该类提供了客户端请求的简化方法和对现有系统类方法的委托调用。
> 2:如何使用原型/外观模式？
> 原型模式：
> 当一个系统应该独立于它的产品创建，构成和表示时。
> 当要实例化的类是在运行时刻指定时，例如，通过动态装载。
> 为了避免创建一个与产品类层次平行的工厂类层次时。
> 当一个类的实例只能有几个不同状态组合中的一种时。建立相应数目的原型并克隆它们可能比每次用合适的状态手工实例化该类更方便一些。
> 外观模式：
> 客户端不需要知道系统内部的复杂联系，整个系统只需提供一个"接待员"即可。
> 定义系统的入口。
> 3:原型/外观模式的优缺点？
> 原型模式：通过复制已有对象来创建新对象。在 Objective-C 中对应 `NSCopying` 协议，需实现 `copyWithZone:` 方法。
> 优点：直接拷贝已有实例，省去重复的初始化逻辑，创建成本低。
> 缺点：当对象内部包含引用类型成员时，需要仔细区分深拷贝与浅拷贝；为已有的复杂类补充正确的拷贝逻辑往往并不容易。
> 外观模式
> 优点：减少系统相互依赖、提高灵活性、提高了安全性。
> 缺点：不符合开闭原则，如果要改东西很麻烦，继承重写都不合适。

- 工厂模式;

> 1:何为工厂模式？
> 这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
> 在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。
> 2:如何使用工厂模式？
> 我们明确地计划不同条件下创建不同实例时。
> 作为一种创建类模式，在任何需要生成复杂对象的地方，都可以使用工厂方法模式。有一点需要注意的地方就是复杂对象适合使用工厂模式，而简单对象，特别是只需要通过 new 就可以完成创建的对象，无需使用工厂模式。如果使用工厂模式，就需要引入一个工厂类，会增加系统的复杂度。
> 3:工厂模式的优缺点？
> 优点：
> 一个调用者想创建一个对象，只要知道其名称就可以了。
> 扩展性高，如果想增加一个产品，只要扩展一个工厂类就可以。
> 屏蔽产品的具体实现，调用者只关心产品的接口。
> 缺点：
> 每次增加一个产品时，都需要增加一个具体类和对象实现工厂，使得系统中类的个数成倍增加，在一定程度上增加了系统的复杂度，同时也增加了系统具体类的依赖。这并不是什么好事。

- 桥接模式;

> 1:何为桥接模式？
> 桥接（Bridge）是用于把抽象化与实现化解耦，使得二者可以独立变化。这种类型的设计模式属于结构型模式，它通过提供抽象化和实现化之间的桥接结构，来实现二者的解耦。
> 这种模式涉及到一个作为桥接的接口，使得实体类的功能独立于接口实现类。这两种类型的类可被结构化改变而互不影响。
> 2:如何使用桥接模式？
> 在有多种可能会变化的情况下，用继承会造成类爆炸问题，扩展起来不灵活。
> 实现系统可能有多个角度分类，每一种角度都可能变化。
> 把这种多角度分类分离出来，让它们独立变化，减少它们之间耦合。
> 3:桥接模式的优缺点？
> 优点 ：抽象和实现的分离、优秀的扩展能力、实现细节对客户透明。
> 缺点：桥接模式的引入会增加系统的理解与设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计与编程。

- 代理模式;

> 1:何为代理模式？
> 在代理模式（Proxy Pattern）中，一个类代表另一个类的功能。这种类型的设计模式属于结构型模式。
> 在代理模式中，我们创建具有现有对象的对象，以便向外界提供功能接口。
> 2:如何使用代理模式？
> 在直接访问对象时带来的问题，比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因（比如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问），直接访问会给使用者或者系统结构带来很多麻烦，我们可以在访问此对象时加上一个对此对象的访问层。
> 想在访问一个类时做一些控制。
> 3:代理模式的优缺点？
> 优点：
> 职责清晰、高扩展性、智能化。
> 缺点：
> 由于在客户端和真实主题之间增加了代理对象，因此有些类型的代理模式可能会造成请求的处理速度变慢。
> 实现代理模式需要额外的工作，有些代理模式的实现非常复杂。

- 单例模式;

> 1:何为单例模式？
> 这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。
> 注意：
> 单例类只能有一个实例。
> 单例类必须自己创建自己的唯一实例。
> 单例类必须给所有其他对象提供这一实例。
> 2：如何使用单例模式？
> 当您想控制实例数目，节省系统资源的时候。
> 3：单例模式的优缺点？
> 优点：
> 在内存里只有一个实例，减少了内存的开销，尤其是频繁的创建和销毁实例（比如管理学院首页页面缓存）。
> 避免对资源的多重占用比如写文件操作。
> 缺点：
> 没有接口，不能继承，与单一职责原则冲突，一个类应该只关心内部逻辑，而不关心外面怎么样来实例化。

- 备忘录模式;

> 1:何为备忘录模式？
> 备忘录模式（Memento Pattern）保存一个对象的某个状态，以便在适当的时候恢复对象。备忘录模式属于行为型模式。
> 2：如何使用备忘录模式？
> 很多时候我们总是需要记录一个对象的内部状态，这样做的目的就是为了允许用户取消不确定或者错误的操作，能够恢复到他原先的状态，使得他有"后悔药"可吃。
> 3：备忘录模式的优缺点？
> 优点：
> 给用户提供了一种可以恢复状态的机制，可以使用户能够比较方便地回到某个历史的状态。
> 实现了信息的封装，使得用户不需要关心状态的保存细节。
> 缺点：
> 消耗资源。如果类的成员变量过多，势必会占用比较大的资源，而且每一次保存都会消耗一定的内存。

- 生成器模式；

> 1：何为生成器模式？
> 建造者模式（Builder Pattern）使用多个简单的对象一步一步构建成一个复杂的对象。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
> 2：如何使用生成器模式？
> 主要解决在软件系统中，有时候面临着"一个复杂对象"的创建工作，其通常由各个部分的子对象用一定的算法构成；由于需求的变化，这个复杂对象的各个部分经常面临着剧烈的变化，但是将它们组合在一起的算法却相对稳定。
> 一些基本部件不会变，而其组合经常变化的时候。
> 3：生成器模式的优缺点？
> 优点：
> 建造者独立，易扩展。
> 便于控制细节风险。
> 缺点：
> 产品必须有共同点，范围有限制。
> 如内部变化复杂，会有很多的建造类。

- 命令模式;

> 1:何为命令模式？
> 命令模式（Command Pattern）是一种数据驱动的设计模式，它属于行为型模式。请求以命令的形式包裹在对象中，并传给调用对象。调用对象寻找可以处理该命令的合适的对象，并把该命令传给相应的对象，该对象执行命令。
> 主要解决的问题？
> 在软件系统中，行为请求者与行为实现者通常是一种紧耦合的关系，但某些场合，比如需要对行为进行记录、撤销或重做、事务等处理时，这种无法抵御变化的紧耦合的设计就不太合适。
> 2：如何使用命令模式？
> 在某些场合，比如要对行为进行"记录、撤销/重做、事务"等处理，这种无法抵御变化的紧耦合是不合适的。在这种情况下，如何将"行为请求者"与"行为实现者"解耦？将一组行为抽象为对象，可以实现二者之间的松耦合。
> 3：命令模式的优缺点？
> 优点：降低了系统耦合度，新的命令可以很容易添加到系统中去。
> 缺点：使用命令模式可能会导致某些系统有过多的具体命令类。

### 12.8 多线程有哪些？如何保证多线程中读写分离，加锁方案？

> NSThread GCD NSOperation

> iOS 实现线程加锁有很多种方式。@synchronized、 NSLock、NSRecursiveLock、NSCondition、NSConditionLock、pthread_mutex、dispatch_semaphore、OSSpinLock、atomic(property) set/ge等等各种方式。

### 12.9 如何删除单链表中一个元素？

> 先来看看删除的原理：因为数据结构是单链表，要想删除第i个节点，就要找到第i个节点；要想找到第i个节点，就要找到第i-1个节点；要想找到第i-1个节点，就要找到第i-2个节点…于是就要从第一个节点开始找起，一直找到第i-1个节点。如何找？让一个指针从头结点开始移动，一直移动到第i-1个节点为止。这个过程中可以用一个变量j从0开始计数，一直自增到i-1。之后呢？我们把第i-1个节点找到了，就让它的指针域指向第i+1个节点，这样就达到了删除的目的。而第i+1个节点的地址又从第i个节点获得，第i个节点的地址又是第i-1个节点的后继。因此我们可以这样做：先让一个指针指向第i-1个节点的后继，（保存i+1节点的地址），再让i-1节点的后继指向第i个节点的后继，这样就将第i个节点删除了。(p->next=q->next;)

```
Status ListDelete(LinkList *L,int i,ElemType *e){
    int j;
    LinkList p,q;
    p = *L; // 声明一结点p指向链表第一个结点
    j = 1;
    while (p->next && j < i)  /* 遍历寻找第i个元素 */
    {
        p = p->next;
        ++j;
    }
    if (!(p->next) || j > i)
        return ERROR;           /* 第i个元素不存在 */
    q = p->next;
    p->next = q->next;            /* 将q的后继赋值给p的后继 */
    *e = q->data;               /* 将q结点中的数据给e */
    free(q);                    /* 让系统回收此结点，释放内存 */
    return OK;
}

```

### 12.10 NSNotificationCenter通知中心的实现原理？

> NSNotificationCenter是类似一个广播中心站，使用defaultCenter来获取应用中的通知中心，它可以向应用任何地方发送和接收通知。在通知中心注册观察者，发送者使用通知中心广播时，以NSNotification的name和object来确定需要发送给哪个观察者。为保证观察者能接收到通知，所以应先向通知中心注册观察者，接着再发送通知这样才能在通知中心调度表中查找到相应观察者进行通知。

```
-(void)postNotification:(NSNotification *)notification;
-(void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
-(void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;

```

> 发送通知通过name和object来确定来标识观察者,name和object两个参数的规则相同即当通知设置name为kChangeNotifition时，那么只会发送给符合name为kChangeNotifition的观察者，同理object指发送给某个特定对象通知，如果只设置了name，那么只有对应名称的通知会触发。如果同时设置name和object参数时就必须同时符合这两个条件的观察者才能接收到通知。

```
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block NS_AVAILABLE(10_6, 4_0);

```

> 第一种方式是比较常用的添加Oberver的方式，接到通知时执行aSelector。
> 第二种方式是基于Block来添加观察者，往通知中心的调度表中添加观察者，这个观察者包括一个queue和一个block,并且会返回这个观察者对象。当接到通知时执行block所在的线程为添加观察者时传入的queue参数，queue也可以为nil，那么block就在通知所在的线程同步执行。
> 这里需要注意的是如果使用第二种的方式创建观察者需要弱引用可能引起循环引用的对象,避免内存泄漏。

---

> NSNotificatinonCenter实现原理:
> NSNotificatinonCenter是使用观察者模式来实现的用于跨层传递消息，用来降低耦合度。
> NSNotificatinonCenter用来管理通知，将观察者注册到NSNotificatinonCenter的通知调度表中，然后发送通知时利用标识符name和object识别出调度表中的观察者，然后调用相应的观察者的方法，即传递消息（在Objective-C中对象调用方法，就是传递消息，消息有name或者selector，可以接受参数，而且可能有返回值），如果是基于block创建的通知就调用NSNotification的block。

### 12.11 推送如何实现的？

- 1.由App向iOS设备发送一个注册通知，用户需要同意系统发送推送。
- 2.iOS应用向APNS远程推送服务器发送App的Bundle Id和设备的UDID。
- 3.APNS根据设备的UDID和App的Bundle Id生成deviceToken再发回给App。
- 4.App再将deviceToken发送给远程推送服务器(自己的服务器), 由服务器保存在数据库中。
- 5.当自己的服务器想发送推送时, 在远程推送服务器中输入要发送的消息并选择发给哪些用户的deviceToken，由远程推送服务器发送给APNS。
- 6.APNS根据deviceToken发送给对应的用户。

### 12.12 SEL的使用和原理？

> SEL 类成员方法的指针
> 可以理解 @selector()就是取类方法的编号,他的行为基本可以等同C语言的中函数指针,只不过C语言中，可以把函数名直接赋给一个函数指针，而Object-C的类不能直接应用函数指针，这样只能做一个@selector语法来取.
> 它的结果是一个SEL类型。这个类型本质是类方法的编号(函数地址)

### 12.13 点击事件如何穿透透明的View?

```
- (UIView*)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    UIView *hitView = [super hitTest:point withEvent:event];
    if(hitView == self){
        return nil;
    }
    return hitView;
}

```

### 12.14 RunLoop的实现原理？

> RunLoop实际上是一个对象，这个对象在循环中用来处理程序运行过程中出现的各种事件（比如说触摸事件、UI刷新事件、定时器事件、Selector事件）和消息，从而保持程序的持续运行，而且在没有事件处理的时候，会进入睡眠模式，从而节省CPU资源,提高程序性能。

**[参考：深入理解Runloop](https://blog.ibireme.com/2015/05/18/runloop/#base)**

### 12.15 简述Runtime，发送消息的过程；

> 动态的添加对象的成员变量和方法;动态交换两个方法的实现;拦截并替换方法;在方法上增加额外功能;实现NSCoding的自动归档和解档;实现字典转模型的自动转换;

> clang -rewrite-objc main.m 查看最终生成代码

消息转发：

- 1：动态方法解析

```
+ (BOOL)resolveInstanceMethod:(SEL)selector;
+ (BOOL)resolveClassMethod:(SEL)selector;

```

- 2：备援接收者(重定向)

```
- (id)forwardingTargetForSelector:(SEL)selector;

```

- 3：完整的消息转发(NSInvocation)

```
- (void)forwardInvocation:(NSInvocation *)invocation;

```

### 12.16 简述weak的实现原理；

> weak 关键字的作用弱引用，所引用对象的计数器不会加一，并在引用对象被释放的时候自动被设置为 nil;
>
> weak是有Runtime维护的weak表;
>
> 3.weak释放为nil过程
> weak被释放为nil，需要对对象整个释放过程了解，如下是对象释放的整体流程：
> 1、调用objc_release
> 2、因为对象的引用计数为0，所以执行dealloc
> 3、在dealloc中，调用了_objc_rootDealloc函数
> 4、在_objc_rootDealloc中，调用了object_dispose函数
> 5、调用objc_destructInstance
> 6、最后调用objc_clear_deallocating。
>
> 对象准备释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。
>
> 其实Weak表是一个hash（哈希）表，然后里面的key是指向对象的地址，Value是Weak指针的地址的数组。
> ###总结
> weak是Runtime维护了一个hash(哈希)表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

### 12.17 写一个单例；

```
+ (instancetype)sharedInstance {
    static RMF *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });

    return instance;
}

```

### 12.18 如何从字符串中得到一个整数？

最简单的方式是使用 Foundation 提供的转换方法（注意 `intValue` 遇到非数字会停止解析、无法区分"0"与非法输入）：

```
NSString *str = @"12345";
NSInteger value = [str integerValue];   // 12345
```

如果需要更严格的控制，可以手动逐字符累加，并处理正负号与非法字符：

```
// 返回 YES 表示解析成功，结果通过 result 输出
- (BOOL)parseInteger:(NSString *)text result:(NSInteger *)result {
    NSInteger value = 0;
    NSInteger sign = 1;
    NSInteger i = 0;
    NSInteger len = text.length;
    if (len == 0) return NO;

    unichar first = [text characterAtIndex:0];
    if (first == '-' || first == '+') {
        sign = (first == '-') ? -1 : 1;
        i = 1;
        if (len == 1) return NO; // 只有符号
    }
    for (; i < len; i++) {
        unichar c = [text characterAtIndex:i];
        if (c < '0' || c > '9') return NO; // 含非数字字符
        value = value * 10 + (c - '0');
    }
    if (result) *result = value * sign;
    return YES;
}
```

### 12.19 数组去重方式；

- 数组法

```
for (NSString *item in originalArr) {
	if (![resultArrM containsObject:item]) {
	  [resultArrM addObject:item];
	}
}

```

- 利用NSDictionary

```
for (NSNumber *n in originalArr) {
	[dict setObject:n forKey:n];
}

```

- NSSet

```
NSSet *set = [NSSet setWithArray:originalArr];

```

### 12.20 实现多个网络请求ABC执行完再执行D

> 方案1：使用group和semaphore
> 方案2：group_enter和group_leave也可以实现
> 下面使用方案1实现例子

```
	dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    dispatch_group_async(group, queue, ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            //异步执行A
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    });

    dispatch_group_async(group, queue, ^{
                dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
             //异步执行B
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    });

    dispatch_group_async(group, queue, ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
             //异步执行C
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    });

    dispatch_group_notify(group, queue, ^{
			 //执行D
    });

```

### 12.21 列表页性能优化

- 如何检测

> 1）Instruments中：Core Animation；
> 2）FPS：CADisplayLink

- 优化方案

> 1、文本、布局计算，提前计算缓存；
> 2、对象创建；CALayer代替UIView;
> 3、离屏渲染；
> 4、图片解码；

> （离屏渲染是指图层在被显示之前是在当前屏幕缓冲区以外开辟的一个缓冲区进行渲染操作。
> 离屏渲染需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上又需要将上下文环境从离屏切换到当前屏幕，而上下文环境的切换是一项高开销的动作。）
>
> 1.阴影（UIView.layer.shadowOffset/shadowRadius/…）
> 2.圆角（当 UIView.layer.cornerRadius 和 UIView.layer.masksToBounds 一起使用时）
> 3.图层蒙板
> 4.开启光栅化（shouldRasterize = true）

> 1、使用CAShapeLayer和UIBezierPath设置圆角;
> 2、UIBezierPath和Core Graphics框架画出一个圆角;

```
//1、使用CAShapeLayer和UIBezierPath设置圆角;
UIBezierPath *maskPath = [UIBezierPath bezierPathWithRoundedRect:imageView.bounds byRoundingCorners:UIRectCornerAllCorners cornerRadii:imageView.bounds.size];
CAShapeLayer *maskLayer = [[CAShapeLayer alloc]init];
maskLayer.frame = imageView.bounds;
maskLayer.path = maskPath.CGPath;
imageView.layer.mask = maskLayer;

//2、UIBezierPath和Core Graphics框架画出一个圆角
UIGraphicsBeginImageContextWithOptions(imageView.bounds.size,NO,1.0);
[[UIBezierPath bezierPathWithRoundedRect:imageView.boundscornerRadius:imageView.frame.size.width]addClip];
[imageView drawRect:imageView.bounds];
imageView.image=UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
[self.view addSubview:imageView];

```

### 12.22 HTTPS

> HTTP+对称加密和非对称加密+证书认证

### 12.23 音视频相关

采集视频,音频–》使用iOS原生框架 AVFoundation.framework
视频滤镜处理–》使用iOS原生框架 CoreImage.framework；使用第三方框架 GPUImage.framework

> CoreImage 与 GPUImage 框架比较:
> 在实际项目开发中,开发者更加倾向使用于GPUImage框架.
> 首先它在使用性能上与iOS提供的原生框架,并没有差别;其次它的使用便利性高于iOS原生框架,最后也是最重要的GPUImage框架是开源的。理解 GPUImage 的封装思路通常需要 OpenGL ES 基础。

视频\音频编码压缩
视频: 使用FFmpeg,X264算法把视频原数据YUV/RGB编码成H264
音频: 使用fdk_aac 将音频数据PCM转换成AAC
视频: VideoToolBox框架
音频: AudioToolBox 框架
硬编码
软编码
推流
流媒体协议: RTMP\RTSP\HLS\FLV
视频封装格式: TS\FLV
音频封装格式: Mp3\AAC
推流: 将采集的音频.视频数据通过流媒体协议发送到流媒体服务器
推流技术
流媒体服务器
数据分发
截屏
实时转码
内容检测
拉流
拉流: 从流媒体服务器中获取音频\视频数据
流媒体协议: RTMP\RTSP\HLS\FLV
音视频解码
视频: 使用FFmpeg,X264算法解码
音频: 使用fdk_aac 解码
视频: VideoToolBox框架
音频: AudioToolBox 框架
硬解码
软解码
播放
ijkplayer,kxmovie 都是基于FFmpeg框架封装的
ijkplayer 播放框架
kxmovie 播放框架


### 12.24 统计一个字符数组中每个字符出现的次数？

```
void main()
{
    char str[20];
    int i,num[256]={0};
    printf("please input string:");
    scanf("%s",str);
    for(i=0;i<strlen(str);i++)
        num[(int)str[i]]++;
    for(i=0;i<256;i++)
        if(num[i]!=0)
            printf("字符%c出现%d次\n",(char)i,num[i]);
}

```

### 12.25 实现一个反转二叉树；

```
@interface TreeNode : NSObject
@property (nonatomic, assign) NSInteger val;
@property (nonatomic, strong) TreeNode *left;
@property (nonatomic, strong) TreeNode *right;
@end
- (void)exchangeNode:(TreeNode *)node {

    //判断是否存在node节点
    if(node) {
        //交换左右节点
        TreeNode *temp = node.left;
        node.left = node.right;
        node.right = temp;
    }

}

- (TreeNode *)invertTree:(TreeNode *)root
{
    //边界条件 递归结束或输入为空情况
    if(!root) {
       return root;
    }

    //递归左右子树
    [self invertTree:root.left];
    [self invertTree:root.right];
    //交换左右子节点
    [self exchangeNode:root];

    return root;
}

```

### 12.26 如何获取VC上所有的Button?

> 递归

### 12.27 排序算法有哪些？

> 冒泡、快速、插入、选择、希尔、堆等等

### 12.28 self和super区别；

> self调用自己方法，super调用父类方法
>
> self 是每个方法都隐含的参数，指向当前消息的接收者（对象本身）；super 并不是一个指针，而是一个编译器指示符——它告诉编译器：消息仍然发给当前对象（self），但方法的查找要从父类的方法列表开始。
>
> 【self class】和【super class】输出是一样的
>
> 1.当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找，然后调用父类的这个方法。
>
> 2.当使用 self 调用时，会使用 objc_msgSend 函数： id objc_msgSend(id theReceiver, SEL theSelector, …)。第 一个参数是消息接收者，第二个参数是调用的具体类方法的 selector，后面是 selector 方法的可变参数。以 [self setName:] 为例，编译器会替换成调用 objc_msgSend 的函数调用，其中 theReceiver 是 self，theSelector 是 @selector(setName:)，这个 selector 是从当前 self 的 class 的方法列表开始找的 setName，当找到后把对应的 selector 传递过去。
>
> 3.当使用 super 调用时，会使用 objc_msgSendSuper 函数：id objc_msgSendSuper(struct objc_super *super, SEL op, …)第一个参数是个objc_super的结构体，第二个参数还是类似上面的类方法的selector,

```
struct objc_super {
      id receiver;
      Class superClass;
};

```

> 当编译器遇到 [super setName:] 时，开始做这几个事：
> 1）构 建 objc_super 的结构体，此时这个结构体的第一个成员变量 receiver 就是 子类，和 self 相同。而第二个成员变量 superClass 就是指父类
> 调用 objc_msgSendSuper 的方法，将这个结构体和 setName 的 sel 传递过去。
> 2）函数里面在做的事情类似这样：从 objc_super 结构体指向的 superClass 的方法列表开始找 setName 的 selector，找到后再以 objc_super->receiver 去调用这个 selector

### 12.29 UIViewController的生命周期；

> initWithCoder; awakeFromNib; loadView; viewDidLoad; viewWillAppear; viewWillLayoutSubviews; viewDidLayoutSubviews; viewDidAppear; viewWillDisappear; viewDidDisappear; dealloc; didReceiveMemoryWarning

### 12.30 UIButton的继承链，如何改变它的点击区域；

> UIButton > UIControl > UIView > UIResponder > NSObject

```
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent*)event
{
    CGRect bounds = self.bounds;
    CGFloat width = MAX(100 - bounds.size.width, 0);
    CGFloat height = MAX(100 - bounds.size.height, 0);
    bounds = CGRectInset(bounds, -width/2, -height/2);
    return CGRectContainsPoint(bounds, point);
}

```

### 12.31 Category

> Build Phases ->Compile Sources 中的编译顺序

> 1.属性。Property
> 2.实例变量。Ivar（属性是给成员变量默认添加了setter和getter方法。tips：如果不用@dynamic修饰的话。）
> 3.isa指针。在Objective-C中，任何类的定义都是对象。类和类的实例（对象）没有任何本质上的区别。任何对象都有isa指针。但是分类没有。

> category 它是在运行期决议的。 因为在运行期即编译完成后，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的。

> 使用Runtime技术中的关联对象可以为类别添加属性。
> 其原因是：关联对象都由AssociationsManager管理，AssociationsManager里面是由一个静态AssociationsHashMap来存储所有的关联对象的。这相当于把所有对象的关联对象都存在一个全局map里面。而map的的key是这个对象的指针地址（任意两个不同对象的指针地址一定是不同的），而这个map的value又是另外一个AssociationsHashMap，里面保存了关联对象的kv对。
> 如何清理关联对象？
> runtime的销毁对象函数objc_destructInstance里面会判断这个对象有没有关联对象，如果有，会调用_object_remove_assocations做关联对象的清理工作。（详见Runtime的源码）

> extension在编译期决议;extension可以添加实例变量，而category是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）

### 12.32 实现setter方法

```
-(void)setName:(NSString *)name{

    if (_name != name) {
        [_name release];
        _name = [name copy];
    }
}

```

### 12.33 iOS判断一个字符串中是否都是数字

```
第一种方式是使用NSScanner：
1\. 整形判断
- (BOOL)isPureInt:(NSString *)string{
NSScanner* scan = [NSScanner scannerWithString:string];
int val;
return [scan scanInt:&val] && [scan isAtEnd];
}

2.浮点形判断：
- (BOOL)isPureFloat:(NSString *)string{
NSScanner* scan = [NSScanner scannerWithString:string];
float val;
return [scan scanFloat:&val] && [scan isAtEnd];
}
第二种方式是使用循环判断
- (BOOL)isPureNumandCharacters:(NSString *)text
{
    for(int i = 0; i < [text length]; ++i) {
        int a = [text characterAtIndex:i];
        if ([self isNum:a]){
            continue;
        } else {
            return NO;
        }
    }
    return YES;
}
或者 C语言中常用的方式.
- (BOOL)isAllNum:(NSString *)string{
    unichar c;
    for (int i=0; i<string.length; i++) {
        c=[string characterAtIndex:i];
        if (!isdigit(c)) {
            return NO;
        }
    }
    return YES;
}
第三种方式则是使用NSString的trimming方法
- (BOOL)isPureNumandCharacters:(NSString *)string
{
string = [string stringByTrimmingCharactersInSet;[NSCharacterSet decimalDigitCharacterSet]];
if(string.length > 0)
{
     return NO;
}
return YES;
}

```

### 12.34 如何合并两个有序数组？

> 比较相邻的两个元素，类似冒泡排序；

```
	NSMutableArray *A = [NSMutableArray arrayWithObjects:@4,@5,@8,@10,@15, nil];
    NSMutableArray *B = [NSMutableArray arrayWithObjects:@2,@6,@7,@9,@11,@12,@13, nil];
    NSMutableArray *C = [NSMutableArray array];
    int count = (int)A.count+(int)B.count;
    int index = 0;
    for (int i = 0; i < count; i++) {
        if (A[0]<B[0]) {
            [C addObject:A[0]];
            [A removeObject:A[0]];
        }
        else if (B[0]<A[0]) {
            [C addObject:B[0]];
            [B removeObject:B[0]];
        }
        if (A.count==0) {
            [C addObjectsFromArray:B];
            index = i+1;
            return;
        }
        else if (B.count==0) {
            [C addObjectsFromArray:A];
            index = i+1;
            return;
        }
    }
    //(2).
    //时间复杂度
    //T(n) = O(f(n)):用"T(n)"表示，"O"为数学符号，f(n)为同数量级，一般是算法中频度最大的语句频度。
    //时间复杂度:T(n) = O(index);

```

### 12.35 GET和POST区别

> TCP 位于传输层，IP 位于网络层，HTTP 位于应用层。
>
> 需要先澄清两个常见误区：① "GET 产生一个 TCP 数据包、POST 产生两个 TCP 数据包" 的说法并不准确——是否分多次发送由数据大小和具体实现决定，与 HTTP 方法无必然关系；POST 的 `100-continue` 也只是可选机制，并非固有行为。② HTTP 规范本身并未限制 GET 的编码方式，URL 长度限制来自浏览器与服务器的实现，而非协议规定。
>
> GET 与 POST 的本质区别在于**语义**：GET 用于获取资源，被定义为安全（不改变服务器状态）且幂等的方法；POST 用于提交数据，通常会改变服务器状态，不保证幂等。
>
> 在常见实践中：GET 参数通过 URL 传递、会出现在浏览器历史与日志中，因此不宜传递敏感信息；POST 参数放在请求体（Request body）中，相对不易直接暴露（但仍需 HTTPS 才能保证传输安全）。
> GET请求会被浏览器主动cache，而POST不会，除非手动设置。
> GET产生的URL地址可以被Bookmark，而POST不可以。
> GET在浏览器回退时是无害的，而POST会再次提交请求。

### 12.36 面向对象编程的六大原则

- 1.单一职责:

> 不要存在多于一个导致类变更的原因。通俗的说，即一个类只负责一项职责。

- 2.里氏替换原则:

> 所有引用基类的地方必须能透明地使用其子类的对象，也就是说子类可以扩展父类的功能，但不能改变父类原有的功能

- 3.依赖倒置:

> 高层模块不应该依赖低层模块，二者都应该依赖其抽象；抽象不应该依赖细节；细节应该依赖抽象。简单的说就是尽量面向接口编程.

- 4.接口隔离:

> 客户端不应该依赖它不需要的接口；一个类对另一个类的依赖应该建立在最小的接口上。接口最小化,过于臃肿的接口依据功能,可以将其拆分为多个接口.

- 5.迪米特法则:

> 一个对象应该对其他对象保持最少的了解,简单的理解就是高内聚,低耦合，一个类尽量减少对其他对象的依赖，并且这个类的方法和属性能用私有的就尽量私有化.

- 6.开闭原则:

> 一个软件实体如类、模块和函数应该对扩展开放，对修改关闭.当软件需求变化时，尽量通过扩展软件实体的行为来实现变化，而不是通过修改已有的代码来实现变化.

### 12.37 iOS Animation

- Core Animation

> CAAnimation作为虚基类实现了CAMediaTiming协议（其实还实现了CAAction协议）。CAAnimation有三个子类CAAnimationGroup（组动画）、CAPropertyAnimation（属性动画）、CATransition（渐变动画）。CAAnimation不能直接使用，应该使用它的子类。作为CAPropertyAnimation也有两个子类CABasicAnimation（基础动画）、CAKeyFrameAnimation（关键帧动画）。CAPropertyAnimation也不能直接使用，应该使用两个子类。综上所述要使用核心动画，可以使用的就是以下四个类（CAAnimationGroup、CATransition、CABasicAnimation、CAKeyFrameAnimation）。

```
1.CABasicAnimation简单使用
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self.view];
    CABasicAnimation *positionAnimation = [CABasicAnimation animationWithKeyPath:@"position"];
    positionAnimation.fromValue = [NSValue valueWithCGPoint:self.testLayer.presentationLayer.position];
    positionAnimation.toValue = [NSValue valueWithCGPoint:point];
    positionAnimation.duration = 1.f;//动画时长
    positionAnimation.removedOnCompletion = NO;//是否在完成时移除
    positionAnimation.fillMode = kCAFillModeForwards;//动画结束后是否保持状态
    [self.testLayer addAnimation:positionAnimation forKey:@"positionAnimation"];
}

2\. CATransition（过渡动画）
	CATransition *transition = [CATransition animation];
    transition.startProgress = 0;//开始进度
    transition.endProgress = 1;//结束进度
    transition.type = kCATransitionReveal;//过渡类型
    transition.subtype = kCATransitionFromLeft;//过渡方向
    transition.duration = 1.f;
    UIColor *color = [UIColor colorWithRed:arc4random_uniform(255) / 255.0 green:arc4random_uniform(255) / 255.0 blue:arc4random_uniform(255) / 255.0 alpha:1.f];
    self.testLayer.backgroundColor = color.CGColor;
    [self.testLayer addAnimation:transition forKey:@"transition"];

 3.CAKeyFrameAnimation关键帧动画
 	CAKeyframeAnimation *moveAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    moveAnimation.path = bezirePath;
    moveAnimation.fillMode = kCAFillModeForwards;
    moveAnimation.removedOnCompletion = NO;
    moveAnimation.duration = 3.f;

 4.AnimationGroup（组动画)
    CAAnimationGroup *groupAnimation = [[CAAnimationGroup alloc] init];
    groupAnimation.animations = @[xScaleAnimation, yScaleAnimation];//将所有动画添加到动画组

```

### 12.38 为什么必须在主线程中操作UI

> 因为UIKit不是线程安全的。试想下面这几种情况：
> 两个线程同时设置同一个背景图片，那么很有可能因为当前图片被释放了两次而导致应用崩溃。
> 两个线程同时设置同一个UIView的背景颜色，那么很有可能渲染显示的是颜色A，而此时在UIView逻辑树上的背景颜色属性为B。
> 两个线程同时操作view的树形结构：在线程A中for循环遍历并操作当前View的所有subView，然后此时线程B中将某个subView直接删除，这就导致了错乱还可能导致应用崩溃。iOS 4 之后，部分绘图 API 以及 UIColor、UIFont 等类已可在非主线程安全使用，但 UIKit 整体仍非线程安全，因此仍强烈建议将 UI 操作放在主线程中执行。

### 12.39 显式和隐式动画的区别

> 隐式动画：直接设置 CALayer 的可动画属性（如 position、opacity 等）时，Core Animation 自动创建的动画。它默认存在，可通过事务（CATransaction）关闭。
>
> 显式动画：开发者显式构造 CAAnimation 对象（如 CABasicAnimation、CAKeyframeAnimation），并通过 `addAnimation:forKey:` 添加到图层上的动画。
>
> 需要注意：UIView 默认关闭了其根 layer 的隐式动画，因此直接修改 UIView.layer 属性不会自动产生动画；而 UIView 的 block 动画（`animateWithDuration:animations:`）本质上是对图层动画的封装，不宜简单地与"显式/隐式"概念画等号。

### 12.40 OC内联函数 inline

> 优点相比于函数:
> 1.inline函数避免了普通函数的,在汇编时必须调用call的缺点:取消了函数的参数压栈，减少了调用的开销,提高效率.所以执行速度确比一般函数的执行速度要快.
> 2)集成了宏的优点,使用时直接用代码替换(像宏一样);

> 优点相比于宏:
> 1.避免了宏的缺点:需要预编译.因为inline内联函数也是函数,不需要预编译.
> 2)编译器在调用一个内联函数时，会首先检查它的参数类型，保证调用正确。然后进行一系列相关检查，就像对待任何一个真正的函数一样，从而消除了宏的隐患和局限性。
>
> 补充：`inline` 本身源自 C/C++，并非 Objective-C 专有；在 OC 中通常以 `static inline` 形式定义工具函数。

> inline内联函数的说明
> 1.内联函数只是我们向编译器提供的申请,编译器不一定采取inline形式调用函数.
> 2.内联函数不能承载大量的代码.如果内联函数的函数体过大,编译器会自动放弃内联.
> 3.内联函数内不允许使用循环语句或开关语句.
> 4.内联函数的定义须在调用之前.

```
//数组添加非空对象
static inline void photoComponentSafeAddObject(NSMutableArray *array, id object) {
    if (array && object) {
        [array addObject:object];
    }
}

static inline CGSize liteVideoImageSizeScaleWithSize(CGSize scaleSize, CGSize pixelSize) {
    CGFloat width = pixelSize.width;
    CGFloat height = pixelSize.height;
    float verticalRadio   = scaleSize.height*1.0/height;
    float horizontalRadio = scaleSize.width*1.0/width;
    float radio = 1;
    if (verticalRadio < 1 || horizontalRadio < 1) {
        radio = verticalRadio < horizontalRadio ? verticalRadio : horizontalRadio;
    }
    width = width*radio;
    height = height*radio;
    // 返回新的改变大小后的size
    return CGSizeMake(width, height);
}
```


### 12.41 SDWebImage原理

一个为UIImageView提供一个分类来支持远程服务器图片加载的库。

#### 12.41.1 功能概览

```
      1、一个添加了web图片加载和缓存管理的UIImageView分类
      2、一个异步图片下载器
      3、一个异步的内存加磁盘综合存储图片并且自动处理过期图片
      4、支持动态gif图
      5、支持webP格式的图片
      6、后台图片解压处理
      7、确保同样的图片url不会下载多次
      8、确保伪造的图片url不会重复尝试下载
      9、确保主线程不会阻塞
```

#### 12.41.2 工作流程

```
1、入口 setImageWithURL:placeholderImage:options: 会先把 placeholderImage 显示，然后 SDWebImageManager 根据 URL 开始处理图片。

2、进入 SDWebImageManager-downloadWithURL:delegate:options:userInfo:，交给 SDImageCache 从缓存查找图片是否已经下载 queryDiskCacheForKey:delegate:userInfo:.

3、先从内存图片缓存查找是否有图片，如果内存中已经有图片缓存，SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo: 到 SDWebImageManager。

4、SDWebImageManagerDelegate 回调 webImageManager:didFinishWithImage: 到 UIImageView+WebCache 等前端展示图片。

5、如果内存缓存中没有，生成 NSInvocationOperation 添加到队列开始从硬盘查找图片是否已经缓存。

6、根据 URLKey 在硬盘缓存目录下尝试读取图片文件。这一步是在 NSOperation 进行的操作，所以回主线程进行结果回调 notifyDelegate:。

7、如果上一操作从硬盘读取到了图片，将图片添加到内存缓存中（如果空闲内存过小，会先清空内存缓存）。SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo:。进而回调展示图片。

8、如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，需要下载图片，回调 imageCache:didNotFindImageForKey:userInfo:。

9、共享或重新生成一个下载器 SDWebImageDownloader 开始下载图片。

10、图片下载由 NSURLConnection 来做，实现相关 delegate 来判断图片下载中、下载完成和下载失败。

11、connection:didReceiveData: 中利用 ImageIO 做了按图片下载进度加载效果。connectionDidFinishLoading: 数据下载完成后交给 SDWebImageDecoder 做图片解码处理。

12、图片解码处理在一个 NSOperationQueue 完成，不会拖慢主线程 UI。如果有需要对下载的图片进行二次处理，最好也在这里完成，效率会好很多。

13、在主线程 notifyDelegateOnMainThreadWithInfo: 宣告解码完成，imageDecoder:didFinishDecodingImage:userInfo: 回调给 SDWebImageDownloader。imageDownloader:didFinishWithImage: 回调给 SDWebImageManager 告知图片下载完成。

14、通知所有的 downloadDelegates 下载完成，回调给需要的地方展示图片。将图片保存到 SDImageCache 中，内存缓存和硬盘缓存同时保存。写文件到硬盘也在以单独 NSInvocationOperation 完成，避免拖慢主线程。

15、SDImageCache 在初始化的时候会注册一些消息通知，在内存警告或退到后台的时候清理内存图片缓存，应用结束的时候清理过期图片。

16、SDWI 也提供了 UIButton+WebCache 和 MKAnnotationView+WebCache，方便使用。

17、SDWebImagePrefetcher 可以预先下载图片，方便后续使用。
```

#### 12.41.3 源码结构

#### 12.41.4 核心对象

#### 12.41.5 图片下载

**1、 SDWebImageDownloader**
- 1.单例，图片下载器，负责图片异步下载，并对图片加载做了优化处理

- 2.图片的下载操作放在一个NSOperationQueue并发操作队列中，队列默认最大并发数是6

- 3.每个图片对应一些回调（下载进度，完成回调等），回调信息会存在downloader的URLCallbacks（一个字典，key是url地址，value是图片下载回调数组）中，URLCallbacks可能被多个线程访问，所以downloader把下载任务放在一个barrierQueue中，并设置屏障保证同一时间只有一个线程访问URLCallbacks。，在创建回调URLCallbacks的block中创建了一个NSOperation并添加到NSOperationQueue中。

- 4.每个图片下载都是一个operation类，创建后添加到一个队列中，SDWebimage定义了一个协议 SDWebImageOperation作为图片下载操作的基础协议，声明了一个cancel方法，用于取消操作。
```
@protocol SDWebImageOperation <NSObject>
-(void)cancel;
@end
```

- 5.对于图片的下载，SDWebImageDownloaderOperation完全依赖于NSURLConnection类，继承和实现了NSURLConnectionDataDelegate协议的方法

```
connection:didReceiveResponse:
connection:didReceiveData:
connectionDidFinishLoading:
connection:didFailWithError:
connection:willCacheResponse:
connectionShouldUseCredentialStorage:
-connection:willSendRequestForAuthenticationChalleng
-connection:didReceiveData:方法，接受数据，创建一个CGImageSourceRef对象，在首次获取数据时（图片width，height），图片下载完成之前，使用CGImageSourceRef对象创建一个图片对象，经过缩放、解压操作生成一个UIImage对象供回调使用，同时还有下载进度处理。
注：缩放：SDWebImageCompat中SDScaledImageForKey函数
 解压：SDWebImageDecoder文件中decodedImageWithImage

```

**2、SDWebImageDownloaderOperation**

- 1.继承自NSOperation类，没有简单实现main方法，而是采用更加灵活的start方法，以便自己管理下载的状态

- 2.start方法中创建了下载使用的NSURLConnections对象，开启了图片的下载，并抛出一个下载开始的通知，

- 3.小结：下载的核心是利用NSURLSession加载数据，每个图片的下载都有一个operation操作来完成，并将这些操作放到一个操作队列中，这样可以实现图片的并发下载。

**3、SDWebImageDecoder（异步对图片进行解码）**

#### 12.41.6 缓存
减少网络流量，下载完图片后存储到本地，下载再获取同一张图片时，直接从本地获取，提升用户体验，能快速从本地获取呈现给用户。
SDWebImage提供了对图片进行了缓存，主要由SDImageCache完成。该类负责处理内存缓存以及一个可选的磁盘缓存，其中磁盘缓存的写操作是异步的，不会对UI造成影响。

**1、内存缓存及磁盘缓存**
- 1.内存缓存的处理由NSCache对象实现，NSCache类似一个集合的容器，它存储key-value对，类似于nsdictionary类，我们通常使用缓存来临时存储短时间使用但创建昂贵的对象，重用这些对象可以优化性能，同时这些对象对于程序来说不是紧要的，如果内存紧张就会自动释放。

- 2.磁盘缓存的处理使用NSFileManager对象实现，图片存储的位置位于cache文件夹，另外SDImageCache还定义了一个串行队列来异步存储图片。

- 3.SDImageCache提供了大量方法来缓存、获取、移除及清空图片。对于图片的索引，我们通过一个key来索引，在内存中，我们将其作为NSCache的key值，而在磁盘中，我们用这个key值作为图片的文件名，对于一个远程下载的图片其url实作为这个key的最佳选择。

**2、存储图片**
先在内存中放置一份缓存，如果需要缓存到磁盘，将磁盘缓存操作作为一个task放到串行队列中处理，会先检查图片格式是jpeg还是png，将其转换为响应的图片数据，最后吧数据写入磁盘中（文件名是对key值做MD5后的串）

**3、查询图片**
内存和磁盘查询图片API：
```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key;
- (UIImage *)imageFromDiskCacheForKey:(NSString *)key;

```
查看本地是否存在key指定的图片，使用一下API：

```
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock;
```
**4、移除图片**
移除图片API：

```
- (void)removeImageForKey:(NSString *)key;
- (void)removeImageForKey:(NSString *)key withCompletion:(SDWebImageNoParamsBlock)completion;
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk;
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion;

```

**5、清理图片（磁盘）**

清空磁盘图片可以选择完全清空和部分清空，完全清空就是吧缓存文件夹删除。

```
- (void)clearDisk;
- (void)clearDiskOnCompletion:(SDWebImageNoParamsBlock)completion;
```
部分清理 会根据设置的一些参数移除部分文件，主要有两个指标：文件的缓存有效期（maxCacheAge：默认是1周）和最大缓存空间大小（maxCacheSize：如果所有文件大小大于最大值，会按照文件最后修改时间的逆序，以每次一半的递归来移除哪些过早的文件，知道缓存文件总大小小于最大值），具体代码参考- (void)cleanDiskWithCompletionBlock；

**6、小结**
SDImageCache处理提供以上API，还提供了获取缓存大小，缓存中图片数量等API，
常用的接口和属性：

```
（1）-getSize  ：获得硬盘缓存的大小

（2）-getDiskCount ： 获得硬盘缓存的图片数量

（3）-clearMemory  ： 清理所有内存图片

（4）- removeImageForKey:(NSString *)key  系列的方法 ： 从内存、硬盘按要求指定清除图片

（5）maxMemoryCost  ：  保存在存储器中像素的总和

（6）maxCacheSize  ：  最大缓存大小 以字节为单位。默认没有设置，也就是为0，而清理磁盘缓存的先决条件为self.maxCacheSize > 0，所以0表示无限制。

（7）maxCacheAge ： 在内存缓存保留的最长时间以秒为单位计算，默认是一周

```

#### 12.41.7 SDWebImageManager

实际使用中并不直接使用SDWebImageDownloader和SDImageCache类对图片进行下载和存储，而是使用SDWebImageManager来管理。包括平常使用UIImageView+WebCache等控件的分类，都是使用SDWebImageManager来处理，该对象内部定义了一个图片下载器（SDWebImageDownloader）和图片缓存（SDImageCache）

```
@interface SDWebImageManager : NSObject

@property (weak, nonatomic) id <SDWebImageManagerDelegate> delegate;

@property (strong, nonatomic, readonly) SDImageCache *imageCache;
@property (strong, nonatomic, readonly) SDWebImageDownloader *imageDownloader;

...

@end
```
SDWebImageManager声明了一个delegate属性，其实是一个id<SDWebImageManagerDelegate>对象，代理声明了两个方法

```
// 控制当图片在缓存中没有找到时，应该下载哪个图片
- (BOOL)imageManager:(SDWebImageManager *)imageManager shouldDownloadImageForURL:(NSURL *)imageURL;

// 允许在图片已经被下载完成且被缓存到磁盘或内存前立即转换
- (UIImage *)imageManager:(SDWebImageManager *)imageManager transformDownloadedImage:(UIImage *)image withURL:(NSURL *)imageURL;
```

这两个方法会在SDWebImageManager的-downloadImageWithURL:options:progress:completed:方法中调用，而这个方法是SDWebImageManager类的核心所在（具体看源码）

SDWebImageManager的几个API：

```
(1）- (void)cancelAll   ： 取消runningOperations中所有的操作，并全部删除

（2）- (BOOL)isRunning  ：检查是否有操作在运行，这里的操作指的是下载和缓存组成的组合操作

（3） - downloadImageWithURL:options:progress:completed:   核心方法

（4）- (BOOL)diskImageExistsForURL:(NSURL *)url  ：指定url的图片是否进行了磁盘缓存

```
#### 12.41.8 视图扩展

在使用SDWebImage的时候，使用最多的是UIImageView+WebCache中的针对UIImageView的扩展，核心方法是sd_setImageWithURL:placeholderImage:options:progress:completed:，   其使用SDWebImageManager单例对象下载并缓存图片。

除了扩展UIImageView外，SDWebImage还扩展了UIView，UIButton，MKAnnotationView等视图类，具体可以参考源码，除了可以使用扩展的方法下载图片，同时也可以使用SDWebImageManager下载图片。

UIView+WebCacheOperation分类：
把当前view对应的图片操作对象存储起来（通过运行时设置属性），在基类中完成
存储的结构：一个loadOperationKey属性，value是一个字典（字典结构： key：UIImageViewAnimationImages或者UIImageViewImageLoad，value是  operation数组（动态图片）或者对象）

UIButton+WebCache分类
会根据不同的按钮状态，下载的图片根据不同的状态进行设置
imageURLStorageKey:{state:url}

#### 12.41.9 技术点

- 1.dispatch_barrier_sync函数，用于对操作设置顺序，确保在执行完任务后再确保后续操作。常用于确保线程安全性操作
- 2.NSMutableURLRequest：用于创建一个网络请求对象，可以根据需要来配置请求报头等信息
- 3.NSOperation及NSOperationQueue：操作队列是OC中一种高级的并发处理方法，基于GCD实现，相对于GCD来说，操作队列的优点是可以取消在任务处理队列中的任务，另外在管理操作间的依赖关系方面容易一些，对SDWebImage中我们看到如何使用依赖将下载顺序设置成后进先出的顺序
- 4.NSURLSession：用于网络请求及相应处理
- 5.开启后台任务
- 6.NSCache类：一个类似于集合的容器，存储key-value对，这一点类似于nsdictionary类，我们通常用使用缓存来临时存储短时间使用但创建昂贵的对象。重用这些对象可以优化性能，因为它们的值不需要重新计算。另外一方面，这些对象对于程序来说不是紧要的，在内存紧张时会被丢弃
- 7.清理缓存图片的策略：特别是最大缓存空间大小的设置。如果所有缓存文件的总大小超过这一大小，则会按照文件最后修改时间的逆序，以每次一半的递归来移除那些过早的文件，直到缓存的实际大小小于我们设置的最大使用空间。
- 8.图片解压操作：这一操作可以查看SDWebImageDecoder.m中+decodedImageWithImage方法的实现。
- 9.对GIF图片的处理
- 10.对WebP图片的处理。

***
### 12.42 什么是Block？
***

- **Block是将函数及其执行上下文封装起来的对象。**

比如：

```
NSInteger num = 3;
    NSInteger(^block)(NSInteger) = ^NSInteger(NSInteger n){
        return n*num;
    };

    block(2);

```

通过clang -rewrite-objc WYTest.m命令编译该.m文件，发现该block被编译成这个形式:

```
    NSInteger num = 3;

    NSInteger(*block)(NSInteger) = ((NSInteger (*)(NSInteger))&__WYTest__blockTest_block_impl_0((void *)__WYTest__blockTest_block_func_0, &__WYTest__blockTest_block_desc_0_DATA, num));

    ((NSInteger (*)(__block_impl *, NSInteger))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 2);

```

其中WYTest是文件名，blockTest是方法名，这些可以忽略。
其中__WYTest__blockTest_block_impl_0结构体为

```
struct __WYTest__blockTest_block_impl_0 {
  struct __block_impl impl;
  struct __WYTest__blockTest_block_desc_0* Desc;
  NSInteger num;
  __WYTest__blockTest_block_impl_0(void *fp, struct __WYTest__blockTest_block_desc_0 *desc, NSInteger _num, int flags=0) : num(_num) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```

__block_impl结构体为

```
struct __block_impl {
  void *isa;//isa指针，所以说Block是对象
  int Flags;
  int Reserved;
  void *FuncPtr;//函数指针
};

```

block内部有isa指针，所以说其本质也是OC对象
block内部则为:

```
static NSInteger __WYTest__blockTest_block_func_0(struct __WYTest__blockTest_block_impl_0 *__cself, NSInteger n) {
  NSInteger num = __cself->num; // bound by copy

        return n*num;
    }

```

所以说 Block是将函数及其执行上下文封装起来的对象
既然block内部封装了函数，那么它同样也有参数和返回值。

#### 12.42.1 Block变量截获

**1、局部变量截获 是值截获。 比如:**

```
    NSInteger num = 3;

    NSInteger(^block)(NSInteger) = ^NSInteger(NSInteger n){

        return n*num;
    };

    num = 1;

    NSLog(@"%zd",block(2));

```

这里的输出是6而不是2，原因就是对局部变量num的截获是值截获。
同样，在block里如果修改变量num，也是无效的，甚至编译器会报错。

**2、局部静态变量截获 是指针截获。**

```
   static  NSInteger num = 3;

    NSInteger(^block)(NSInteger) = ^NSInteger(NSInteger n){

        return n*num;
    };

    num = 1;

    NSLog(@"%zd",block(2));

```

输出为2，意味着num = 1这里的修改num值是有效的，即是指针截获。
同样，在block里去修改变量m，也是有效的。

**3、全局变量，静态全局变量截获：不截获,直接取值。**

我们同样用clang编译看下结果。

```
static NSInteger num3 = 300;

NSInteger num4 = 3000;

- (void)blockTest
{
    NSInteger num = 30;

    static NSInteger num2 = 3;

    __block NSInteger num5 = 30000;

    void(^block)(void) = ^{

        NSLog(@"%zd",num);//局部变量

        NSLog(@"%zd",num2);//静态变量

        NSLog(@"%zd",num3);//全局变量

        NSLog(@"%zd",num4);//全局静态变量

        NSLog(@"%zd",num5);//__block修饰变量
    };

    block();
}

```

编译后

```
struct __WYTest__blockTest_block_impl_0 {
  struct __block_impl impl;
  struct __WYTest__blockTest_block_desc_0* Desc;
  NSInteger num;//局部变量
  NSInteger *num2;//静态变量
  __Block_byref_num5_0 *num5; // by ref//__block修饰变量
  __WYTest__blockTest_block_impl_0(void *fp, struct __WYTest__blockTest_block_desc_0 *desc, NSInteger _num, NSInteger *_num2, __Block_byref_num5_0 *_num5, int flags=0) : num(_num), num2(_num2), num5(_num5->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```

（ impl.isa = &_NSConcreteStackBlock;这里注意到这一句，即说明该block是栈block）
可以看到局部变量被编译成值形式，而静态变量被编成指针形式，全局变量并未截获。而__block修饰的变量也是以指针形式截获的，并且生成了一个新的结构体**对象**：

```
struct __Block_byref_num5_0 {
  void *__isa;
__Block_byref_num5_0 *__forwarding;
 int __flags;
 int __size;
 NSInteger num5;
};

```

该对象有个属性：num5，即我们用__block修饰的变量。
这里__forwarding是指向自身的(栈block)。
一般情况下，如果我们要对block截获的局部变量进行赋值操作需添加__block
修饰符，而对全局变量，静态变量是不需要添加__block修饰符的。
另外，block里访问self或成员变量都会去截获self。

#### 12.42.2 Block的几种形式

- **分为全局Block(_NSConcreteGlobalBlock)、栈Block(_NSConcreteStackBlock)、堆Block(_NSConcreteMallocBlock)三种形式**

    **其中栈Block存储在栈(stack)区，堆Block存储在堆(heap)区，全局Block存储在已初始化数据(.data)区**

**1、不使用外部变量的block是全局block**

比如：

```
    NSLog(@"%@",[^{
        NSLog(@"globalBlock");
    } class]);

```

输出：

```
__NSGlobalBlock__

```

**2、使用外部变量并且未进行copy操作的block是栈block**

比如:

```
  NSInteger num = 10;
    NSLog(@"%@",[^{
        NSLog(@"stackBlock:%zd",num);
    } class]);

```

输出：

```
__NSStackBlock__

```

日常开发常用于这种情况:

```
[self testWithBlock:^{
    NSLog(@"%@",self);
}];

- (void)testWithBlock:(dispatch_block_t)block {
    block();

    NSLog(@"%@",[block class]);
}

```

**3、对栈block进行copy操作，就是堆block，而对全局block进行copy，仍是全局block**

- 比如堆1中的全局进行copy操作，即赋值：

```
void (^globalBlock)(void) = ^{
        NSLog(@"globalBlock");
    };

 NSLog(@"%@",[globalBlock class]);

```

输出：

```
__NSGlobalBlock__

```

仍是全局block

- 而对2中的栈block进行赋值操作：

```
NSInteger num = 10;

void (^mallocBlock)(void) = ^{

        NSLog(@"stackBlock:%zd",num);
    };

NSLog(@"%@",[mallocBlock class]);

```

输出：

```
__NSMallocBlock__

```

对栈blockcopy之后，并不代表着栈block就消失了，左边的mallock是堆block，右边被copy的仍是栈block
比如:

```
[self testWithBlock:^{

    NSLog(@"%@",self);
}];

- (void)testWithBlock:(dispatch_block_t)block
{
    block();

    dispatch_block_t tempBlock = block;

    NSLog(@"%@,%@",[block class],[tempBlock class]);
}

```

输出：

```
__NSStackBlock__,__NSMallocBlock__

```

- **即如果对栈Block进行copy，将会copy到堆区，对堆Block进行copy，将会增加引用计数，对全局Block进行copy，因为是已经初始化的，所以什么也不做。**

另外，__block变量在copy时，由于__forwarding的存在，栈上的__forwarding指针会指向堆上的__forwarding变量，而堆上的__forwarding指针指向其自身，所以，如果对__block的修改，实际上是在修改堆上的__block变量。

**即__forwarding指针存在的意义就是，无论在任何内存位置， 都可以顺利地访问同一个__block变量。**

- 另外由于block捕获的__block修饰的变量会去持有变量，那么如果用__block修饰self，且self持有block，并且block内部使用到__block修饰的self时，就会造成多循环引用，即self持有block，block 持有__block变量，而__block变量持有self，造成内存泄漏。
    比如:

```
  __block typeof(self) weakSelf = self;

    _testBlock = ^{

        NSLog(@"%@",weakSelf);
    };

    _testBlock();

```

如果要解决这种循环引用，可以主动断开__block变量对self的持有，即在block内部使用完weakself后，将其置为nil，但这种方式有个问题，如果block一直不被调用，那么循环引用将一直存在。
所以，我们最好还是用__weak来修饰self

***
### 12.43 RunLoop剖析
***

**RunLoop是通过内部维护的`事件循环(Event Loop)`来对`事件/消息进行管理`的一个对象。**

1. 没有消息处理时，休眠以避免资源占用，由用户态切换到内核态([CPU-内核态和用户态](https://www.jianshu.com/p/3bb1cdd44ef0))
2. 有消息需要处理时，立刻被唤醒，由内核态切换到用户态

**为什么main函数不会退出？**

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

UIApplicationMain内部默认开启了主线程的RunLoop，并执行了一段无限循环的代码（不是简单的for循环或while循环）

```
//无限循环代码模式(伪代码)
int main(int argc, char * argv[]) {
    BOOL running = YES;
    do {
        // 执行各种任务，处理各种事件
        // ......
    } while (running);

    return 0;
}

```

UIApplicationMain函数一直没有返回，而是不断地接收处理消息以及等待休眠，所以运行程序之后会保持持续运行状态。

#### 12.43.1 RunLoop的数据结构

`NSRunLoop(Foundation)`是`CFRunLoop(CoreFoundation)`的封装，提供了面向对象的API
RunLoop 相关的主要涉及五个类：

`CFRunLoop`：RunLoop对象
`CFRunLoopMode`：运行模式
`CFRunLoopSource`：输入源/事件源
`CFRunLoopTimer`：定时源
`CFRunLoopObserver`：观察者

**1、CFRunLoop**

由`pthread`(线程对象，说明RunLoop和线程是一一对应的)、`currentMode`(当前所处的运行模式)、`modes`(多个运行模式的集合)、`commonModes`(模式名称字符串集合)、`commonModelItems`(Observer,Timer,Source集合)构成

**2、CFRunLoopMode**

由name、source0、source1、observers、timers构成

**3、CFRunLoopSource**

分为source0和source1两种

- `source0`:
    即非基于port的，也就是用户触发的事件。需要手动唤醒线程，将当前线程从内核态切换到用户态
- `source1`:
    基于port的，包含一个 mach_port 和一个回调，可监听系统端口和通过内核和其他线程发送的消息，能主动唤醒RunLoop，接收分发系统事件。
    具备唤醒线程的能力

**4、CFRunLoopTimer**

基于时间的触发器，基本上说的就是NSTimer。在预设的时间点唤醒RunLoop执行回调。因为它是基于RunLoop的，因此它不是实时的（就是NSTimer 是不准确的。 因为RunLoop只负责分发源的消息。如果线程当前正在处理繁重的任务，就有可能导致Timer本次延时，或者少执行一次）。

**5、CFRunLoopObserver**

监听以下时间点:`CFRunLoopActivity`

- `kCFRunLoopEntry`
    RunLoop准备启动
- `kCFRunLoopBeforeTimers`
    RunLoop将要处理一些Timer相关事件
- `kCFRunLoopBeforeSources`
    RunLoop将要处理一些Source事件
- `kCFRunLoopBeforeWaiting`
    RunLoop将要进行休眠状态,即将由用户态切换到内核态
- `kCFRunLoopAfterWaiting`
    RunLoop被唤醒，即从内核态切换到用户态后
- `kCFRunLoopExit`
    RunLoop退出
- `kCFRunLoopAllActivities`
    监听所有状态

**6、各数据结构之间的联系**

线程和RunLoop一一对应， RunLoop和Mode是一对多的，Mode和source、timer、observer也是一对多的

![](https://upload-images.jianshu.io/upload_images/17495317-4664eb028a3953a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 12.43.2 RunLoop的Mode

关于Mode首先要知道一个RunLoop 对象中可能包含多个Mode，且每次调用 RunLoop 的主函数时，只能指定其中一个 Mode(CurrentMode)。切换 Mode，需要重新指定一个 Mode 。主要是为了分隔开不同的 Source、Timer、Observer，让它们之间互不影响。

![](https://upload-images.jianshu.io/upload_images/17495317-14318d5787a24b75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当RunLoop运行在Mode1上时，是无法接受处理Mode2或Mode3上的Source、Timer、Observer事件的

总共是有五种`CFRunLoopMode`:

- `kCFRunLoopDefaultMode`：默认模式，主线程是在这个运行模式下运行

- `UITrackingRunLoopMode`：跟踪用户交互事件（用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他Mode影响）

- `UIInitializationRunLoopMode`：在刚启动App时第进入的第一个 Mode，启动完成后就不再使用

- `GSEventReceiveRunLoopMode`：接受系统内部事件，通常用不到

- `kCFRunLoopCommonModes`：伪模式，不是一种真正的运行模式，是同步Source/Timer/Observer到多个Mode中的一种解决方案

#### 12.43.3 RunLoop的实现机制

![](https://upload-images.jianshu.io/upload_images/17495317-9ef6bad15230df46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这张图在网上流传比较广。
对于RunLoop而言最核心的事情就是保证线程在没有消息的时候休眠，在有消息时唤醒，以提高程序性能。RunLoop这个机制是依靠系统内核来完成的（苹果操作系统核心组件Darwin中的Mach）。

![](https://upload-images.jianshu.io/upload_images/17495317-50fd6da547027acd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RunLoop通过`mach_msg()`函数接收、发送消息。它的本质是调用函数`mach_msg_trap()`，相当于是一个系统调用，会触发内核状态切换。在用户态调用 `mach_msg_trap()`时会切换到内核态；内核态中内核实现的`mach_msg()`函数会完成实际的工作。
即基于port的source1，监听端口，端口有消息就会触发回调；而source0，要手动标记为待处理和手动唤醒RunLoop

[Mach消息发送机制](https://www.jianshu.com/p/a764aad31847)
大致逻辑为：
1. 通知观察者 RunLoop 即将启动。
2. 通知观察者即将要处理Timer事件。
3. 通知观察者即将要处理source0事件。
4. 处理source0事件。
5. 如果基于端口的源(Source1)准备好并处于等待状态，进入步骤9。
6. 通知观察者线程即将进入休眠状态。
7. 将线程置于休眠状态，由用户态切换到内核态，直到下面的任一事件发生才唤醒线程。

- 一个基于 port 的Source1 的事件(图里应该是source0)。
- 一个 Timer 到时间了。
- RunLoop 自身的超时时间到了。
- 被其他调用者手动唤醒。

8. 通知观察者线程将被唤醒。
9. 处理唤醒时收到的事件。

- 如果用户定义的定时器启动，处理定时器事件并重启RunLoop。进入步骤2。
- 如果输入源启动，传递相应的消息。
- 如果RunLoop被显式唤醒而且时间还没超时，重启RunLoop。进入步骤2

10. 通知观察者RunLoop结束。

#### 12.43.4 RunLoop与NSTimer

一个比较常见的问题：滑动tableView时，定时器还会生效吗？
默认情况下RunLoop运行在`kCFRunLoopDefaultMode`下，而当滑动tableView时，RunLoop切换到`UITrackingRunLoopMode`，而Timer是在`kCFRunLoopDefaultMode`下的，就无法接受处理Timer的事件。
怎么去解决这个问题呢？把Timer添加到`UITrackingRunLoopMode`上并不能解决问题，因为这样在默认情况下就无法接受定时器事件了。
所以我们需要把Timer同时添加到`UITrackingRunLoopMode`和`kCFRunLoopDefaultMode`上。
那么如何把timer同时添加到多个mode上呢？就要用到`NSRunLoopCommonModes`了

```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

Timer就被添加到多个mode上，这样即使RunLoop由`kCFRunLoopDefaultMode`切换到`UITrackingRunLoopMode`下，也不会影响接收Timer事件

#### 12.43.5 RunLoop和线程

- 线程和RunLoop是一一对应的,其映射关系是保存在一个全局的 Dictionary 里
- 自己创建的线程默认是没有开启RunLoop的

**1、怎么创建一个常驻线程？**

1. 为当前线程开启一个RunLoop（第一次调用 [NSRunLoop currentRunLoop]方法时实际是会先去创建一个RunLoop）
1. 向当前RunLoop中添加一个Port/Source等维持RunLoop的事件循环（如果RunLoop的mode中一个item都没有，RunLoop会退出）
2. 启动该RunLoop

```
   @autoreleasepool {
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
```

**2、输出下边代码的执行顺序**

```
 NSLog(@"1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    [self performSelector:@selector(test) withObject:nil afterDelay:10];
    NSLog(@"3");
});
NSLog(@"4");
- (void)test
{
    NSLog(@"5");
}
```

答案是1423，test方法并不会执行。
原因是如果是带afterDelay的延时函数，会在内部创建一个 NSTimer，然后添加到当前线程的RunLoop中。也就是如果当前线程没有开启RunLoop，该方法会失效。
那么我们改成:

```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2");
        [[NSRunLoop currentRunLoop] run];
        [self performSelector:@selector(test) withObject:nil afterDelay:10];
        NSLog(@"3");
    });
```

然而test方法依然不执行。
原因是如果RunLoop的mode中一个item都没有，RunLoop会退出。即在调用RunLoop的run方法后，由于其mode中没有添加任何item去维持RunLoop的时间循环，RunLoop随即还是会退出。
所以我们自己启动RunLoop，一定要在添加item后

```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2");
        [self performSelector:@selector(test) withObject:nil afterDelay:10];
        [[NSRunLoop currentRunLoop] run];
        NSLog(@"3");
    });
```

**3、怎样保证子线程数据回来更新UI的时候不打断用户的滑动操作？**

当我们在子请求数据的同时滑动浏览当前页面，如果数据请求成功要切回主线程更新UI，那么就会影响当前正在滑动的体验。
我们就可以将更新UI事件放在主线程的`NSDefaultRunLoopMode`上执行即可，这样就会等用户不再滑动页面，主线程RunLoop由`UITrackingRunLoopMode`切换到`NSDefaultRunLoopMode`时再去更新UI

```
[self performSelectorOnMainThread:@selector(reloadData) withObject:nil waitUntilDone:NO modes:@[NSDefaultRunLoopMode]];
```


---
