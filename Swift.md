
# 一、Swift 基础

## 1. Swift 是什么？和 Objective-C 有什么区别？

### 答案

Swift 是 Apple 推出的现代编程语言，主要用于 iOS、macOS、watchOS、tvOS 开发。

和 Objective-C 相比：

| 对比项 | Swift | Objective-C |
|---|---|---|
| 类型系统 | 强类型，类型安全 | 动态类型能力更强 |
| 语法 | 简洁现代 | C 风格 + 消息发送 |
| 空值处理 | Optional 强约束 | nil 可以随处传递 |
| 内存管理 | ARC | ARC |
| 函数式特性 | 支持 map/filter/reduce、闭包 | 支持较弱 |
| 泛型 | 原生强泛型 | 主要依赖轻量泛型 |
| 协议能力 | 协议 + 扩展非常强 | 协议能力相对简单 |
| 运行时 | 静态派发更多 | Runtime 动态派发强 |

### 重点

Swift 更安全、更现代，Objective-C 更动态、更依赖 Runtime。

---

## 2. Swift 中 let 和 var 的区别？

### 答案

`let` 定义常量，赋值后不能修改。

```swift
let name = "Tom"
```

`var` 定义变量，可以修改。

```swift
var age = 18
age = 20
```

### 注意

如果 `let` 修饰的是引用类型对象，不能修改引用本身，但对象内部属性是否能改，取决于对象属性是否可变。

```swift
class Person {
    var name = "Tom"
}

let p = Person()
p.name = "Jack"   // 可以
// p = Person()   // 不可以
```

### 原因

`let p` 固定的是引用地址，不是对象内部所有内容。

---

## 3. Swift 中 Optional 是什么？

### 答案

Optional 表示一个值可能存在，也可能不存在。

```swift
var name: String? = "Tom"
name = nil
```

本质上 Optional 是一个枚举：

```swift
enum Optional<Wrapped> {
    case some(Wrapped)
    case none
}
```

### 常见解包方式

```swift
var name: String? = "Tom"

// 强制解包
print(name!)

// 可选绑定
if let value = name {
    print(value)
}

// guard let
guard let value = name else {
    return
}

// nil 合并
let result = name ?? "默认值"

// 可选链
let count = name?.count
```

### 易错点

强制解包 `!` 如果遇到 nil 会崩溃。

```swift
var name: String? = nil
print(name!) // 崩溃
```

---

## 4. if let 和 guard let 有什么区别？

### 答案

`if let` 适合局部使用。

```swift
if let name = name {
    print(name)
}
```

`guard let` 适合提前退出，解包后的变量可以在后续作用域继续使用。

```swift
guard let name = name else {
    return
}

print(name)
```

| 对比项 | if let | guard let |
|---|---|---|
| 使用场景 | 局部判断 | 参数校验、提前返回 |
| 作用域 | if 内部 | guard 后续作用域 |
| 可读性 | 嵌套较多 | 减少嵌套 |

---

## 5. Swift 中 `?`、`!`、`??` 分别是什么意思？

### 答案

`?` 表示 Optional，也可以用于可选链。

```swift
var name: String?
let count = name?.count
```

`!` 表示强制解包，也可以表示隐式解包 Optional。

```swift
print(name!)
var label: UILabel!
```

`??` 是 nil 合并运算符。

```swift
let result = name ?? "默认值"
```

### 易错点

`UILabel!` 本质上仍然是 Optional，只是使用时自动解包，如果是 nil 仍然会崩溃。

---

## 6. Swift 中值类型和引用类型的区别？

### 答案

Swift 中：

- 值类型：`struct`、`enum`、`tuple`
- 引用类型：`class`、`closure`

### 值类型

赋值或传参时会拷贝。

```swift
struct User {
    var name: String
}

var u1 = User(name: "Tom")
var u2 = u1
u2.name = "Jack"

print(u1.name) // Tom
print(u2.name) // Jack
```

### 引用类型

赋值或传参时传递引用。

```swift
class Person {
    var name: String
    
    init(name: String) {
        self.name = name
    }
}

let p1 = Person(name: "Tom")
let p2 = p1
p2.name = "Jack"

print(p1.name) // Jack
print(p2.name) // Jack
```

| 对比项 | struct / enum | class |
|---|---|---|
| 类型 | 值类型 | 引用类型 |
| 内存 | 拷贝值 | 拷贝引用 |
| 继承 | 不支持 | 支持 |
| ARC | 不参与 | 参与 |
| 线程安全 | 相对更安全 | 需要注意共享状态 |

---

## 7. Swift 的 struct 和 class 怎么选择？

### 答案

优先使用 `struct`，需要引用语义时使用 `class`。

### 使用 struct 的场景

数据模型、配置对象、轻量数据封装。

```swift
struct User {
    let id: Int
    var name: String
}
```

### 使用 class 的场景

需要继承、身份标识、共享状态、生命周期管理。

```swift
class ViewModel {
    var data: [String] = []
}
```

| 场景 | 推荐 |
|---|---|
| 数据不可共享 | struct |
| 数据需要共享修改 | class |
| 需要继承 | class |
| 需要被 NSObject / Runtime 使用 | class |
| 需要线程安全倾向 | struct |

---

## 8. struct 为什么可以有 mutating 方法？

### 答案

struct 是值类型，默认情况下实例方法不能修改自身属性。

如果方法内部需要修改属性，必须加 `mutating`。

```swift
struct Counter {
    var count = 0
    
    mutating func add() {
        count += 1
    }
}
```

### 原因

值类型方法默认认为 `self` 是不可变的。`mutating` 表示这个方法会修改当前值。

### 注意

如果实例是 `let`，即使方法是 `mutating` 也不能调用。

```swift
let counter = Counter()
// counter.add() // 报错
```

---

## 9. Swift 中 String 是值类型还是引用类型？

### 答案

`String` 是值类型，本质是 `struct`。

```swift
var a = "hello"
var b = a
b = "world"

print(a) // hello
print(b) // world
```

Swift 的 `String`、`Array`、`Dictionary`、`Set` 都是值类型，但内部做了写时复制，也就是 Copy On Write。

---

## 10. 什么是 Copy On Write？

### 答案

Copy On Write，简称 COW，意思是写时复制。

多个变量共享同一份底层存储，只有当其中一个变量发生修改时，才真正复制数据。

```swift
var arr1 = [1, 2, 3]
var arr2 = arr1

arr2.append(4)

print(arr1) // [1, 2, 3]
print(arr2) // [1, 2, 3, 4]
```

### 好处

减少不必要的内存复制，提高性能。

### 常见类型

- Array
- Dictionary
- Set
- String
- Data

### 追问

Swift 内部可以通过引用计数判断底层存储是否唯一引用。如果唯一引用，直接修改；如果不是唯一引用，先复制再修改。

---

# 二、闭包

## 11. Swift 中闭包是什么？

### 答案

闭包是可以捕获上下文变量的代码块。

```swift
let block = {
    print("hello")
}

block()
```

带参数和返回值：

```swift
let add: (Int, Int) -> Int = { a, b in
    return a + b
}
```

简写：

```swift
let add: (Int, Int) -> Int = { $0 + $1 }
```

闭包可以作为变量、参数、返回值。

---

## 12. 闭包为什么会造成循环引用？

### 答案

当对象强引用闭包，闭包又强引用对象，就会产生循环引用。

```swift
class ViewController {
    var callback: (() -> Void)?
    
    func setup() {
        callback = {
            self.doSomething()
        }
    }
    
    func doSomething() {}
}
```

引用关系：

```text
ViewController -> callback -> self(ViewController)
```

解决方式：

```swift
callback = { [weak self] in
    self?.doSomething()
}
```

---

## 13. weak 和 unowned 的区别？

### 答案

| 对比项 | weak | unowned |
|---|---|---|
| 是否强引用 | 否 | 否 |
| 是否自动置 nil | 是 | 否 |
| 类型 | Optional | 非 Optional |
| 对象释放后访问 | 返回 nil | 崩溃 |
| 使用场景 | 生命周期不确定 | 生命周期确定比闭包长 |

### weak 示例

```swift
callback = { [weak self] in
    self?.doSomething()
}
```

### unowned 示例

```swift
callback = { [unowned self] in
    self.doSomething()
}
```

大部分业务代码优先使用 `weak`，更安全。

---

## 14. `[weak self]` 后为什么常见 `guard let self = self else { return }`？

### 答案

因为 `[weak self]` 捕获的是 Optional。

```swift
callback = { [weak self] in
    guard let self = self else {
        return
    }
    
    self.doSomething()
}
```

Swift 新写法：

```swift
callback = { [weak self] in
    guard let self else { return }
    self.doSomething()
}
```

这样后续就不用一直写 `self?`。

---

## 15. escaping 和 non-escaping 闭包有什么区别？

### 答案

默认闭包参数是非逃逸闭包。如果闭包可能在函数返回后才执行，需要加 `@escaping`。

### 非逃逸闭包

```swift
func execute(_ block: () -> Void) {
    block()
}
```

### 逃逸闭包

```swift
var callback: (() -> Void)?

func setup(_ block: @escaping () -> Void) {
    callback = block
}
```

### 判断标准

闭包是否被保存起来，或者异步执行。

```swift
func request(completion: @escaping () -> Void) {
    DispatchQueue.main.async {
        completion()
    }
}
```

---

# 三、协议和泛型

## 16. Swift 协议是什么？

### 答案

协议定义一组规范，具体类型去实现这些规范。

```swift
protocol Runnable {
    func run()
}

struct Person: Runnable {
    func run() {
        print("run")
    }
}
```

协议可以要求属性、方法、构造器。

```swift
protocol Named {
    var name: String { get }
}

protocol Buildable {
    init()
}
```

---

## 17. protocol extension 有什么作用？

### 答案

协议扩展可以给协议提供默认实现。

```swift
protocol Runnable {
    func run()
}

extension Runnable {
    func run() {
        print("default run")
    }
}

struct Dog: Runnable {}
Dog().run() // default run
```

好处是提高代码复用，减少重复实现。

---

## 18. 协议扩展中的方法派发有什么坑？

### 答案

如果方法是协议本身声明的，走动态派发。如果方法只写在 extension 中，没有在协议中声明，可能走静态派发。

```swift
protocol Animal {
    func eat()
}

extension Animal {
    func eat() {
        print("Animal eat")
    }
    
    func sleep() {
        print("Animal sleep")
    }
}

struct Dog: Animal {
    func eat() {
        print("Dog eat")
    }
    
    func sleep() {
        print("Dog sleep")
    }
}

let dog = Dog()
dog.eat()    // Dog eat
dog.sleep()  // Dog sleep

let animal: Animal = dog
animal.eat()   // Dog eat
animal.sleep() // Animal sleep
```

### 原因

`sleep()` 没有在协议中声明，只是扩展方法。当变量类型是 `Animal` 时，调用的是协议扩展默认实现。

---

## 19. associatedtype 是什么？

### 答案

`associatedtype` 是协议中的占位类型。

```swift
protocol Container {
    associatedtype Item
    
    func append(_ item: Item)
    func get() -> Item
}
```

实现时确定具体类型：

```swift
struct IntContainer: Container {
    var value: Int = 0
    
    func append(_ item: Int) {
        
    }
    
    func get() -> Int {
        return value
    }
}
```

作用是让协议支持泛型能力。

---

## 20. 泛型有什么作用？

### 答案

泛型可以让代码适配多种类型，同时保持类型安全。

```swift
func swapValue<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var a = 10
var b = 20
swapValue(&a, &b)
```

好处是避免重复代码，保留类型检查。

---

## 21. some 和 any 的区别？

### 答案

`some Protocol` 表示不透明类型。`any Protocol` 表示存在类型。

### some

编译期确定具体类型，只是对外隐藏。

```swift
func makeView() -> some View {
    Text("Hello")
}
```

要求每条返回路径实际类型一致。

```swift
func makeValue(_ flag: Bool) -> some Shape {
    if flag {
        return Circle()
    } else {
        return Circle()
    }
}
```

下面不行：

```swift
func makeValue(_ flag: Bool) -> some Shape {
    if flag {
        return Circle()
    } else {
        return Rectangle()
    }
}
```

### any

运行时可以存放任意符合协议的类型。

```swift
let value: any Runnable = Dog()
```

| 对比项 | some | any |
|---|---|---|
| 含义 | 某个具体类型 | 任意符合协议的类型 |
| 类型确定时间 | 编译期 | 运行期 |
| 性能 | 通常更好 | 有额外开销 |
| 常见场景 | SwiftUI 返回 View | 存储不同协议对象 |

---

# 四、内存管理

## 22. Swift 的内存管理方式是什么？

### 答案

Swift 使用 ARC，也就是自动引用计数，管理 class 实例的生命周期。

当强引用计数变为 0，对象释放。

```swift
class Person {
    deinit {
        print("Person deinit")
    }
}

var p: Person? = Person()
p = nil // 释放
```

ARC 只管理引用类型。struct、enum 是值类型，不通过 ARC 管理生命周期。

---

## 23. strong、weak、unowned 的区别？

### 答案

| 类型 | 是否增加引用计数 | 是否可为 nil | 对象释放后 |
|---|---|---|---|
| strong | 是 | 可以，取决于类型 | 持有对象 |
| weak | 否 | 必须是 Optional | 自动变 nil |
| unowned | 否 | 通常非 Optional | 访问崩溃 |

### weak 示例

```swift
class Person {
    weak var dog: Dog?
}
```

### unowned 示例

```swift
class CreditCard {
    unowned let owner: Customer
    
    init(owner: Customer) {
        self.owner = owner
    }
}
```

---

## 24. Swift 中常见循环引用场景有哪些？

### 答案

### 1. 对象互相强引用

```swift
class A {
    var b: B?
}

class B {
    var a: A?
}
```

解决：

```swift
class B {
    weak var a: A?
}
```

### 2. 闭包强引用 self

```swift
class VC {
    var block: (() -> Void)?
    
    func setup() {
        block = {
            self.loadData()
        }
    }
    
    func loadData() {}
}
```

解决：

```swift
block = { [weak self] in
    self?.loadData()
}
```

### 3. Timer 强引用 target

```swift
timer = Timer.scheduledTimer(
    timeInterval: 1,
    target: self,
    selector: #selector(run),
    userInfo: nil,
    repeats: true
)
```

解决：使用 block timer，并弱引用 self，或者在合适时机 invalidate。

---

## 25. deinit 什么时候调用？

### 答案

当 class 实例没有任何强引用时，系统调用 `deinit`。

```swift
class Person {
    deinit {
        print("释放")
    }
}

var p: Person? = Person()
p = nil
```

如果没有调用 `deinit`，通常说明对象还被强引用，可能存在循环引用或缓存持有。

---

# 五、集合和函数式编程

## 26. Array、Set、Dictionary 的区别？

### 答案

| 类型 | 特点 |
|---|---|
| Array | 有序，可重复 |
| Set | 无序，不重复 |
| Dictionary | 键值对，无序 |

```swift
let arr = [1, 2, 2, 3]
let set: Set = [1, 2, 2, 3]
let dict = ["name": "Tom", "age": "18"]
```

输出：

```swift
arr // [1, 2, 2, 3]
set // [1, 2, 3]
```

---

## 27. map、compactMap、flatMap 的区别？

### 答案

### map

一对一转换。

```swift
let arr = [1, 2, 3]
let result = arr.map { $0 * 2 }
// [2, 4, 6]
```

### compactMap

转换后去掉 nil。

```swift
let arr = ["1", "2", "abc"]
let result = arr.compactMap { Int($0) }
// [1, 2]
```

### flatMap

可以压平嵌套集合。

```swift
let arr = [[1, 2], [3, 4]]
let result = arr.flatMap { $0 }
// [1, 2, 3, 4]
```

---

## 28. filter 和 reduce 的作用？

### 答案

### filter

筛选符合条件的元素。

```swift
let arr = [1, 2, 3, 4]
let result = arr.filter { $0 > 2 }
// [3, 4]
```

### reduce

把集合合并成一个结果。

```swift
let arr = [1, 2, 3, 4]
let sum = arr.reduce(0) { $0 + $1 }
// 10
```

简写：

```swift
let sum = arr.reduce(0, +)
```

---

# 六、错误处理

## 29. Swift 如何处理错误？

### 答案

Swift 使用 `throw`、`try`、`do-catch` 处理错误。

```swift
enum NetworkError: Error {
    case invalidURL
    case noData
}

func request() throws -> String {
    throw NetworkError.noData
}

do {
    let result = try request()
    print(result)
} catch {
    print(error)
}
```

### try? 和 try! 的区别

```swift
let value = try? request()
```

`try?` 会把结果转成 Optional，失败返回 nil。

```swift
let value = try! request()
```

`try!` 表示确定不会失败，失败会崩溃。

---

# 七、访问控制

## 30. Swift 有哪些访问控制？

### 答案

| 关键字 | 作用范围 |
|---|---|
| open | 模块外可访问、可继承、可重写 |
| public | 模块外可访问，不可在模块外继承或重写 |
| internal | 当前模块内可访问，默认级别 |
| fileprivate | 当前文件内可访问 |
| private | 当前声明作用域内可访问 |

### open 和 public 区别

```swift
open class A {
    open func test() {}
}

public class B {
    public func test() {}
}
```

`open` 允许模块外继承和重写。`public` 只允许模块外访问，不允许模块外继承和重写。

---

# 八、初始化

## 31. Swift 中指定构造器和便利构造器区别？

### 答案

指定构造器是主要构造器，必须完成当前类所有存储属性初始化。

```swift
class Person {
    var name: String
    
    init(name: String) {
        self.name = name
    }
}
```

便利构造器用 `convenience` 修饰，必须调用同类中的其他构造器。

```swift
class Person {
    var name: String
    
    init(name: String) {
        self.name = name
    }
    
    convenience init() {
        self.init(name: "Default")
    }
}
```

### 构造器规则

1. 指定构造器必须调用父类指定构造器。
2. 便利构造器必须调用当前类其他构造器。
3. 便利构造器最终必须走到指定构造器。

---

## 32. required init 有什么作用？

### 答案

`required init` 表示子类必须实现该构造器。

```swift
class Animal {
    required init() {}
}

class Dog: Animal {
    required init() {
        super.init()
    }
}
```

常见场景是协议要求构造器。

```swift
protocol Buildable {
    init()
}

class Base: Buildable {
    required init() {}
}
```

---

# 九、派发机制

## 33. Swift 方法派发有哪些方式？

### 答案

Swift 常见方法派发：

| 派发方式 | 特点 |
|---|---|
| 静态派发 | 编译期确定，性能最好 |
| 函数表派发 | class 方法常见 |
| 消息派发 | Objective-C Runtime |
| 协议表派发 | protocol witness table |

struct 方法通常静态派发：

```swift
struct User {
    func test() {}
}
```

class 普通方法通常函数表派发：

```swift
class Person {
    func test() {}
}
```

`@objc dynamic` 使用 Objective-C 消息派发：

```swift
class Person: NSObject {
    @objc dynamic func test() {}
}
```

---

## 34. final 有什么作用？

### 答案

`final` 表示禁止继承或重写。

```swift
final class UserManager {
    
}
```

方法也可以 final：

```swift
class Person {
    final func run() {}
}
```

### 好处

1. 防止被继承或重写。
2. 编译器可以优化派发方式。
3. 代码语义更明确。

---

## 35. static 和 class 的区别？

### 答案

二者都可以定义类型方法。

### static

不能被子类重写。

```swift
class Animal {
    static func run() {}
}
```

### class

可以被子类重写。

```swift
class Animal {
    class func run() {}
}

class Dog: Animal {
    override class func run() {}
}
```

在 struct 和 enum 中只能用 `static`。

---

# 十、并发

## 36. Swift 中 async/await 是什么？

### 答案

`async/await` 是 Swift 的结构化并发语法，让异步代码写起来像同步代码。

```swift
func fetchData() async -> String {
    return "data"
}

Task {
    let data = await fetchData()
    print(data)
}
```

优点是比传统回调更清晰，避免回调嵌套。

---

## 37. Task 是什么？

### 答案

`Task` 表示一个异步任务。

```swift
Task {
    let result = await loadData()
    print(result)
}
```

可以取消：

```swift
let task = Task {
    await loadData()
}

task.cancel()
```

取消不是强制停止，而是协作式取消。任务内部需要检查取消状态。

```swift
try Task.checkCancellation()
```

---

## 38. MainActor 是什么？

### 答案

`MainActor` 表示主线程执行上下文，通常用于 UI 更新。

```swift
@MainActor
class ViewModel {
    var title: String = ""
    
    func updateTitle() {
        title = "Hello"
    }
}
```

或者：

```swift
await MainActor.run {
    self.label.text = "Hello"
}
```

UIKit / SwiftUI 的 UI 更新应该在主线程。

---

## 39. actor 是什么？解决什么问题？

### 答案

`actor` 是 Swift 并发中的引用类型，用于保护可变状态，避免数据竞争。

```swift
actor Counter {
    private var value = 0
    
    func increment() {
        value += 1
    }
    
    func getValue() -> Int {
        return value
    }
}
```

调用 actor 方法需要 `await`：

```swift
let counter = Counter()

Task {
    await counter.increment()
    let value = await counter.getValue()
}
```

作用是同一时间只允许一个任务访问 actor 内部可变状态。

---

## 40. GCD 和 async/await 的区别？

### 答案

| 对比项 | GCD | async/await |
|---|---|---|
| 编程方式 | 队列 + 闭包 | 结构化并发 |
| 可读性 | 容易嵌套 | 类似同步代码 |
| 取消 | 自己管理 | Task 支持取消 |
| 错误处理 | 回调传递 | throws + try await |
| 状态隔离 | 手动保证 | actor / MainActor |

### GCD 示例

```swift
DispatchQueue.global().async {
    let data = load()
    
    DispatchQueue.main.async {
        updateUI(data)
    }
}
```

### async/await 示例

```swift
Task {
    let data = await load()
    await MainActor.run {
        updateUI(data)
    }
}
```

---

# 十一、属性

## 41. 存储属性和计算属性区别？

### 答案

存储属性保存真实数据。

```swift
struct Person {
    var name: String
}
```

计算属性不直接存储值，而是通过 getter/setter 计算。

```swift
struct Circle {
    var radius: Double
    
    var diameter: Double {
        get {
            radius * 2
        }
        set {
            radius = newValue / 2
        }
    }
}
```

---

## 42. lazy 属性是什么？

### 答案

`lazy` 表示延迟初始化，第一次使用时才创建。

```swift
class VC {
    lazy var label: UILabel = {
        let label = UILabel()
        label.text = "Hello"
        return label
    }()
}
```

### 特点

1. 必须用 `var`，不能用 `let`。
2. 第一次访问时才初始化。
3. 初始化闭包中可以使用 `self`。
4. 默认不是线程安全的。

---

## 43. property observer 是什么？

### 答案

属性观察器用于监听属性变化。

```swift
var name: String = "" {
    willSet {
        print("即将设置为 \(newValue)")
    }
    didSet {
        print("之前是 \(oldValue)")
    }
}
```

初始化时不会调用 `willSet` 和 `didSet`。

---

# 十二、KVC / KVO / Runtime 相关

## 44. Swift 支持 KVC / KVO 吗？

### 答案

Swift 原生类型不依赖 Objective-C Runtime。如果要使用 KVC / KVO，通常需要继承 `NSObject`，并使用 `@objc dynamic`。

```swift
class Person: NSObject {
    @objc dynamic var name: String = ""
}
```

### KVO 示例

```swift
var observation: NSKeyValueObservation?

observation = person.observe(\.name, options: [.new, .old]) { object, change in
    print(change)
}
```

`@objc` 暴露给 Objective-C。`dynamic` 强制使用动态派发。

---

## 45. @objc、dynamic、@objcMembers 区别？

### 答案

### @objc

把 Swift 声明暴露给 Objective-C。

```swift
@objc func test() {}
```

### dynamic

强制动态派发。

```swift
@objc dynamic func test() {}
```

### @objcMembers

让类中可暴露的成员自动暴露给 Objective-C。

```swift
@objcMembers
class Person: NSObject {
    var name: String = ""
    func run() {}
}
```

不是所有 Swift 特性都能暴露给 Objective-C，比如泛型、部分 enum、struct 等。

---

# 十三、SwiftUI / Combine

## 46. SwiftUI 中 @State 是什么？

### 答案

`@State` 用于 View 内部私有状态管理。

```swift
struct ContentView: View {
    @State private var count = 0
    
    var body: some View {
        Button("Add") {
            count += 1
        }
    }
}
```

当 `count` 改变时，View 会重新计算 body。

---

## 47. @Binding 是什么？

### 答案

`@Binding` 用于子 View 修改父 View 的状态。

```swift
struct ParentView: View {
    @State private var isOn = false
    
    var body: some View {
        ChildView(isOn: $isOn)
    }
}

struct ChildView: View {
    @Binding var isOn: Bool
    
    var body: some View {
        Toggle("开关", isOn: $isOn)
    }
}
```

`@Binding` 不持有数据，只是建立双向绑定。

---

## 48. @StateObject 和 @ObservedObject 区别？

### 答案

| 属性包装器 | 作用 |
|---|---|
| @StateObject | 当前 View 创建并持有对象 |
| @ObservedObject | 外部传入对象，当前 View 不负责创建 |
| @EnvironmentObject | 从环境中获取共享对象 |

```swift
class ViewModel: ObservableObject {
    @Published var title = "Hello"
}

struct AView: View {
    @StateObject var vm = ViewModel()
    
    var body: some View {
        BView(vm: vm)
    }
}

struct BView: View {
    @ObservedObject var vm: ViewModel
    
    var body: some View {
        Text(vm.title)
    }
}
```

自己创建用 `@StateObject`。外部传入用 `@ObservedObject`。

---

## 49. Combine 中 Publisher、Subscriber、Cancellable 是什么？

### 答案

### Publisher

发布数据。

```swift
let publisher = Just("Hello")
```

### Subscriber

订阅数据。

```swift
publisher.sink { value in
    print(value)
}
```

### Cancellable

管理订阅生命周期。

```swift
var cancellables = Set<AnyCancellable>()

publisher
    .sink { value in
        print(value)
    }
    .store(in: &cancellables)
```

如果不持有 `AnyCancellable`，订阅可能立即释放。

---

# 十四、常见代码题

## 50. 下面代码输出什么？

```swift
struct User {
    var name: String
}

var u1 = User(name: "A")
var u2 = u1
u2.name = "B"

print(u1.name)
print(u2.name)
```

### 答案

```text
A
B
```

### 原因

struct 是值类型，赋值时拷贝。

---

## 51. 下面代码输出什么？

```swift
class User {
    var name: String
    
    init(name: String) {
        self.name = name
    }
}

let u1 = User(name: "A")
let u2 = u1
u2.name = "B"

print(u1.name)
print(u2.name)
```

### 答案

```text
B
B
```

### 原因

class 是引用类型，`u1` 和 `u2` 指向同一个对象。

---

## 52. 下面代码会不会崩溃？

```swift
var name: String? = nil
print(name!)
```

### 答案

会崩溃。因为 `name` 是 nil，强制解包 nil 会触发运行时错误。

---

## 53. 下面代码输出什么？

```swift
var arr1 = [1, 2, 3]
var arr2 = arr1
arr2.append(4)

print(arr1)
print(arr2)
```

### 答案

```text
[1, 2, 3]
[1, 2, 3, 4]
```

### 原因

Array 是值类型，使用写时复制。

---

## 54. 下面代码有没有循环引用？

```swift
class VC {
    var block: (() -> Void)?
    
    func setup() {
        block = {
            self.test()
        }
    }
    
    func test() {}
}
```

### 答案

有。

```text
VC 强引用 block
block 强引用 self
```

解决：

```swift
block = { [weak self] in
    self?.test()
}
```

---

## 55. 下面代码输出什么？

```swift
protocol Animal {
    func eat()
}

extension Animal {
    func eat() {
        print("Animal eat")
    }
    
    func sleep() {
        print("Animal sleep")
    }
}

struct Dog: Animal {
    func eat() {
        print("Dog eat")
    }
    
    func sleep() {
        print("Dog sleep")
    }
}

let dog: Animal = Dog()
dog.eat()
dog.sleep()
```

### 答案

```text
Dog eat
Animal sleep
```

### 原因

`eat()` 在协议中声明，走协议动态派发。`sleep()` 只在协议扩展中定义，变量类型是 `Animal` 时调用默认实现。

---

# 十五、进阶追问题

## 56. Swift 为什么是类型安全语言？

### 答案

Swift 编译器会在编译期检查类型是否匹配，避免很多运行期错误。

```swift
let age: Int = 18
// age = "hello" // 编译错误
```

Optional 也增强了空值安全。

```swift
var name: String? = nil
```

必须显式处理 nil，不能随意当成 String 使用。

---

## 57. Swift 中 enum 为什么强大？

### 答案

Swift 的 enum 支持关联值、原始值、方法、计算属性、协议。

### 关联值

```swift
enum Result {
    case success(String)
    case failure(Error)
}
```

### 原始值

```swift
enum Status: Int {
    case success = 200
    case notFound = 404
}
```

### 方法

```swift
enum Direction {
    case left
    case right
    
    func desc() -> String {
        switch self {
        case .left:
            return "left"
        case .right:
            return "right"
        }
    }
}
```

---

## 58. Result 类型有什么作用？

### 答案

`Result` 用于表示成功或失败。

```swift
func request(completion: (Result<String, Error>) -> Void) {
    completion(.success("data"))
}
```

处理：

```swift
request { result in
    switch result {
    case .success(let data):
        print(data)
    case .failure(let error):
        print(error)
    }
}
```

好处是比单独传 `data` 和 `error` 更清晰。

---

## 59. Swift 中 defer 是什么？

### 答案

`defer` 用于延迟执行代码，在当前作用域结束前执行。

```swift
func test() {
    print("1")
    
    defer {
        print("2")
    }
    
    print("3")
}

test()
```

输出：

```text
1
3
2
```

多个 defer 后进先出。

```swift
defer { print("A") }
defer { print("B") }
```

输出：

```text
B
A
```

常见用途：释放资源、关闭文件、解锁。

---

## 60. inout 是什么？

### 答案

`inout` 允许函数修改外部变量。

```swift
func add(_ value: inout Int) {
    value += 1
}

var num = 10
add(&num)

print(num) // 11
```

调用时必须加 `&`。

---

# 十六、实战设计题

## 61. 如何设计一个网络请求层？

### 答案思路

可以分层：

```text
API 定义层
Request 构建层
NetworkClient 请求层
Response 解析层
Error 统一处理层
业务 Service 层
```

### 示例

```swift
enum APIError: Error {
    case invalidURL
    case noData
    case decodeFailed
    case serverError(Int)
}

protocol APIRequest {
    associatedtype Response: Decodable
    
    var path: String { get }
    var method: String { get }
    var parameters: [String: Any]? { get }
}

final class NetworkClient {
    func send<T: APIRequest>(_ request: T) async throws -> T.Response {
        guard let url = URL(string: request.path) else {
            throw APIError.invalidURL
        }
        
        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = request.method
        
        let (data, response) = try await URLSession.shared.data(for: urlRequest)
        
        if let httpResponse = response as? HTTPURLResponse,
           !(200...299).contains(httpResponse.statusCode) {
            throw APIError.serverError(httpResponse.statusCode)
        }
        
        do {
            return try JSONDecoder().decode(T.Response.self, from: data)
        } catch {
            throw APIError.decodeFailed
        }
    }
}
```

### 重点

1. 错误统一。
2. 请求和解析解耦。
3. 支持泛型 Decodable。
4. 支持 async/await。
5. 方便 Mock 和单元测试。

---

## 62. 如何避免 Swift 项目中的循环引用？

### 答案

重点排查以下场景：

### 1. 闭包

```swift
request { [weak self] result in
    self?.handle(result)
}
```

### 2. delegate

```swift
weak var delegate: SomeDelegate?
```

### 3. Timer

```swift
timer?.invalidate()
timer = nil
```

### 4. Notification

```swift
NotificationCenter.default.removeObserver(self)
```

### 5. Combine

```swift
publisher
    .sink { [weak self] value in
        self?.handle(value)
    }
    .store(in: &cancellables)
```

---

## 63. 如何做 Swift 性能优化？

### 答案

可以从以下方向：

### 1. 减少不必要的拷贝

理解值类型和 COW。

```swift
var arr = largeArray
arr.append(item)
```

### 2. 合理使用 final

```swift
final class UserManager {}
```

减少动态派发成本。

### 3. 避免主线程重任务

```swift
Task.detached {
    let data = heavyWork()
}
```

### 4. 图片和列表优化

UITableView / UICollectionView 中避免重复创建昂贵对象。

### 5. 减少桥接

Swift 和 Objective-C 类型频繁桥接会带来开销。

```swift
String <-> NSString
Array <-> NSArray
Dictionary <-> NSDictionary
```

### 6. 使用 Instruments

排查 Time Profiler、Leaks、Allocations。

---

# 十七、综合必背题

## 64. Swift 为什么推荐 struct？

### 答案

因为 struct 是值类型，有以下优点：

1. 数据隔离更好。
2. 不容易产生循环引用。
3. 线程安全性更好。
4. 编译器优化空间大。
5. 语义清晰。

但并不是所有场景都用 struct。需要共享状态、继承、Objective-C Runtime 能力时，仍然使用 class。

---

## 65. Swift 中协议为什么比继承更常用？

### 答案

协议更灵活，可以让不同类型共享能力，而不要求它们属于同一继承体系。

```swift
protocol Trackable {
    func track()
}

struct ButtonEvent: Trackable {
    func track() {}
}

class PageEvent: Trackable {
    func track() {}
}
```

### 优点

1. 降低耦合。
2. 支持值类型和引用类型。
3. 可以配合 extension 提供默认实现。
4. 更符合组合优于继承的设计思想。

---

## 66. Swift 中什么时候用 weak self，什么时候不用？

### 答案

### 需要 weak self

闭包被当前对象长期持有，或者异步回调生命周期不确定。

```swift
viewModel.callback = { [weak self] in
    self?.reloadData()
}
```

### 可以不用 weak self

闭包不会被长期保存，立即执行。

```swift
UIView.animate(withDuration: 0.3) {
    self.view.alpha = 0
}
```

不是所有闭包都必须写 `[weak self]`。关键看闭包是否可能被 self 间接或直接持有。

---

## 67. Swift 中 Optional 为什么比 Objective-C 的 nil 更安全？

### 答案

Objective-C 中给 nil 发消息不会崩溃，容易掩盖问题。

Swift 中 Optional 要求开发者明确处理 nil。

```swift
var name: String? = nil
```

不能直接这样用：

```swift
// print(name.count)
```

必须解包：

```swift
if let name = name {
    print(name.count)
}
```

好处是很多空值问题在编译期就能暴露。

---

# 十八、最终复习重点

## 必须掌握

1. Optional 原理和解包方式。
2. struct 和 class 区别。
3. 值类型、引用类型、COW。
4. 闭包循环引用。
5. weak 和 unowned。
6. 协议、协议扩展、派发机制。
7. 泛型、associatedtype、some、any。
8. ARC 原理。
9. async/await、Task、actor、MainActor。
10. SwiftUI 中 State、Binding、StateObject。
11. map、compactMap、flatMap、reduce。
12. 初始化规则。
13. final、static、class。
14. @objc、dynamic。
15. 错误处理。

---

# 高频简答模板

## Swift 最大特点是什么？

Swift 是一门类型安全、现代化、支持值类型、协议、泛型、闭包、函数式编程和结构化并发的语言。相比 Objective-C，它语法更简洁，空值处理更安全，编译期检查更多，运行时错误更少。

---

## struct 和 class 核心区别怎么答？

struct 是值类型，赋值和传参会拷贝；class 是引用类型，赋值和传参传递引用。struct 不支持继承，不参与 ARC；class 支持继承，参与 ARC，可能产生循环引用。Swift 中一般优先使用 struct，需要共享状态、继承、Runtime 能力时使用 class。

---

## Optional 怎么答？

Optional 表示一个值可能有值，也可能为 nil。本质是枚举，有 some 和 none 两种情况。Swift 通过 Optional 强制开发者处理空值，避免很多运行时崩溃。常见处理方式有 if let、guard let、??、可选链和强制解包。

---

## 闭包循环引用怎么答？

当对象强引用闭包，闭包内部又强引用 self，就会形成循环引用。解决方式是在闭包捕获列表中使用 `[weak self]` 或 `[unowned self]`。如果对象生命周期不确定，优先使用 weak；如果确定 self 生命周期比闭包长，可以使用 unowned。

---

## actor 怎么答？

actor 是 Swift 并发中的引用类型，用于保护内部可变状态，避免多任务同时访问造成数据竞争。访问 actor 的隔离属性或方法通常需要 await。它通过串行化访问保证状态安全，比手动加锁更符合 Swift 结构化并发模型。
