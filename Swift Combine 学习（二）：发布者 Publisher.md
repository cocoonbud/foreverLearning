# Swift Combine 学习（二）：发布者 Publisher

[TOC]

## 引言

在上一篇文章中，初步简单的介绍了 Combine 框架的基本概念，大概有了一个初印象。本文将开始介绍 Combine 中的发布者（Publisher）。Publisher 是 Combine 框架的核心组件之一，负责生成和传递数据流。通过理解 Publisher 的类型、特性和使用方法，可以更好地在 Combine 中生成和管理数据流。

## 发布者 (`Publisher`)

> Declares that a type can transmit a sequence of values over time.
>
> 声明一个类型可以随着时间推移传输一系列的值。

发布者（`Publisher`）是 Combine 框架中的核心概念之一，它定义如何生成并传递一系列值。像是观察者模式中的 Observable。它可以使用 `operator` 来组合变换，生成新的 `Publisher`。发布者会随时间的推移将一系列值发送给一个或多个订阅者 `Subscriber`。发布者遵循 `Publisher` 协议，该协议定义了两个关联类型：

1. `Output`：发布者发送出的值。
2. `Failure`：发布者可能产生的错误，遵循 `Error` 协议。

```swift
public protocol Publisher<Output, Failure> {
    /// The kind of values published by this publisher.
    /// 此发布者发布的值的类型
    associatedtype Output

    /// The kind of errors this publisher might publish.
    /// 此发布者可能发布的错误类型
    /// Use `Never` if this `Publisher` does not publish errors.
    /// 如果这个发布者不会发错误，用 Never
    associatedtype Failure : Error

    /// Attaches the specified subscriber to this publisher.
    /// 将指定的订阅者附加给此发布者

    /// Implementations of ``Publisher`` must implement this method.
    /// 
    /// The provided implementation of ``Publisher/subscribe(_:)-4u8kn``calls this method.
    ///
    /// - Parameter subscriber: The subscriber to attach to this ``Publisher``, after which it can receive values.
    func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
}

extension Publisher {
    /// Attaches the specified subject to this publisher.
    /// 将指定的 Subject 订阅到一个 Publisher
    /// - Parameter subject: The subject to attach to this publisher.
    public func subscribe<S>(_ subject: S) -> AnyCancellable where S : Subject, Self.Failure == S.Failure, Self.Output == S.Output
}

extension Publisher {
    /// Specifies the scheduler on which to receive elements from the publisher.指定用于接收发布者元素的调度器。
    ///
    /// You use the ``Publisher/receive(on:options:)`` operator to receive results and completion on a specific scheduler, such as performing UI work on the main run loop.你可以使用 ``Publisher/receive(on:options:)`` 操作符在特定的调度器上接收结果和完成信号，比如在主运行循环上执行 UI 工作。
    ///
    /// In contrast with ``Publisher/subscribe(on:options:)``, which affects upstream messages, ``Publisher/receive(on:options:)`` changes the execution context of downstream messages. 与影响上游消息的 ``Publisher/subscribe(on:options:)`` 相比，``Publisher/receive(on:options:)`` 改变的是下游消息的执行上下文。
    ///
    /// - Parameters:
    ///   - scheduler: The scheduler the publisher uses for element delivery.发布者用于传递元素的调度器。
    ///   - options: Scheduler options used to customize element delivery.用于自定义元素传递的调度器选项。
    /// - Returns: A publisher that delivers elements using the specified scheduler.返回一个使用指定调度器传递元素的发布者。
    public func receive<S>(on scheduler: S, options: S.SchedulerOptions? = nil) -> Publishers.ReceiveOn<Self, S> where S : Scheduler
}

... 其他很多 Publisher 的 extension ...
```

`Publisher` 通过 `receive<S>(subscriber: S)` 来接受订阅。

一个发布者可以发布多个值，可以有两种可能的状态：成功或失败。在成功状态下，发布者会发送 `Output` 类型的值；在失败状态下，发布者则会发送 `Failure` 类型的错误。如果发布者不会失败，`Failure` 类型通常会被设置为 `Never`，表示不会产生错误。

Combine 框架内置了很多发布者，包括 `Just`、`Future`、`Deferred`、`Empty`、`Fail`、`Record` 以及 `PassthroughSubject` 和 `curValueSubject`。

一些 `Publisher` 举例：

1. `Future`：

   * 用于表示一个异步操作，该操作最终会产生一个值或一个错误。

   * 在以下例子中模拟一个异步操作，随机决定成功或失败。

     ```Swift
     let futureP = Future<Int, Error> { promise in
         DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
             let success = Bool.random()
             if success {
                 promise(.success(2))
             } else {
                 promise(.failure(NSError(domain: "ExampleError", code: 0, userInfo: nil)))
             }
         }
     }
     ```
   
2. `Just`：

   * 创建一个只发出单个值然后立即完成的 Publisher。

     ```Swift
     let justP = Just("Hello, World!")
     ```

3. `Deferred`：

   * 延迟创建 Publisher 直到有订阅者订阅。

     ```Swift
     let deferredPublisher = Deferred {
         Just("Hello, World!")
     }
     ```

### `ConnectablePublisher`

`ConnectablePublisher` 可以让开发者控制数据流的开始发送时机。通常情况下，`Publisher` 在有订阅者时会立即开始发送数据，但 `ConnectablePublisher` 可以通过调用 `connect()` 来显式启动数据流。这在需要同步多个订阅者的场景中非常有用，例如网络请求或数据库查询。

比如当多个订阅者订阅了同一个非 `ConnectablePublisher` 的 `Publisher`，有可能会出现其中一个订阅者收到了订阅内容，而另外一个订阅者却没收到的情况。这时候就可以使用 使用 `makeConnectable()` 和 `connect()` 控制发布。

```swift
import Combine

class PublisherExample {
    private var cancellables = Set<AnyCancellable>()
    
    func demonstratePublishers() {
        // 1. Publisher
        print("🍎普通 Publisher🍎")
        let numbers = PassthroughSubject<Int, Never>()
        
        // 第一个订阅者
        numbers.map { value -> Int in
            print("处理数据: \(value)")
            return value * 2
        }.sink { value in
            print("订阅者A: \(value)")
        }.store(in: &cancellables)
        
        // 第二个订阅者
        numbers.map { value -> Int in
            print("处理数据: \(value)")
            return value * 2
        }.sink { value in
            print("订阅者B: \(value)")
        }.store(in: &cancellables)
            
        // 发送数据
        numbers.send(1)
        numbers.send(2)
        
        // 2. ConnectablePublisher
        print("\n🍎ConnectablePublisher🍎")
        let subject = PassthroughSubject<Int, Never>()
        let connectablePublisher = subject.map { value -> Int in
            print("处理数据: \(value)")
            return value * 2
        }.makeConnectable()
        
        // 第一个订阅者
        connectablePublisher.sink { value in
            print("订阅者A: \(value)")
        }.store(in: &cancellables)
        
        // 第二个订阅者
        connectablePublisher.sink { value in
            print("订阅者B: \(value)")
        }.store(in: &cancellables)
        
        // 连接发布者
        connectablePublisher.connect().store(in: &cancellables)
        
        // 发送数据
        subject.send(1)
        subject.send(2)
    }
}

let exp = PublisherExample()
exp.demonstratePublishers()

/* 输出:
🍎普通 Publisher🍎
处理数据: 1
订阅者A: 2
处理数据: 1
订阅者B: 2
处理数据: 2
订阅者A: 4
处理数据: 2
订阅者B: 4

🍎ConnectablePublisher🍎
处理数据: 1
订阅者B: 2
订阅者A: 2
处理数据: 2
订阅者B: 4
订阅者A: 4
*/
```

* 普通 Publisher:
  * 每个订阅者订阅时都会触发一次完整的数据流
  * 耗时操作会被执行多次
  * 订阅者获得的是独立的数据流
* ConnectablePublisher:
  * 在调用 connect() 之前不会开始发送数据
  * 所有订阅者共享同一个数据流
  * 耗时操作只执行一次
  * 适合需要等待所有订阅者准备就绪才开始发送数据的场景

### 引用共享

在 Combine 中，引用共享通常是指多个订阅者（`Subscriber`）共享同一个发布者（`Publisher`）的输出，而不是每个订阅者都触发一次数据生成。这可以通过 `.share()` 操作符实现。

关于 `ConnectablePublisher` 和引用共享所使用的 `share` 操作符号，在此先不展开讲解，后面讲到操作符的时候再举例说明。

### Subject

> A publisher that exposes a method for outside callers to publish elements.
> 一个公开了方法供外部调用者发布元素的发布者。

`Subject` 是一个特殊的 `Publisher`。它的特殊之处在于：

1. 主动发送能力：

   1. 普通的 Publisher 只能在创建时定义数据
   2. Subject 通过 send(_:) 方法可以在外部主动发送值到数据流中

2. 控制权的不同：

   1. 普通 Publisher 的数据流是由内部逻辑控制的（比如 Just、Future）
   2. Subject 允许外部代码通过 send 方法控制数据流

   ```Swift
   // 普通 Publisher：只能在创建时定义数据
   let publisher = [1, 2, 3].publisher  // 数据流固定
   
   // Subject：可以随时发送新数据
   let subject = PassthroughSubject<Int, Never>()
   subject.send(1) 
   subject.send(2)
   subject.send(completion: .finished)  // 还可以主动结束
   ```


```Swift
public protocol Subject<Output, Failure> : AnyObject, Publisher {

    /// Sends a value to the subscriber.
    ///
    /// - Parameter value: The value to send.
    func send(_ value: Self.Output)

    /// Sends a completion signal to the subscriber.
    ///
    /// - Parameter completion: A `Completion` instance which indicates whether publishing has finished normally or failed with an error.
    func send(completion: Subscribers.Completion<Self.Failure>)

    /// Sends a subscription to the subscriber.
    ///
    /// This call provides the ``Subject`` an opportunity to establish demand for any new upstream subscriptions.
    ///
    /// - Parameter subscription: The subscription instance through which the subscriber can request elements.
    func send(subscription: any Subscription)
}
```

Combine 内置了两种 `Subject`，分别是 `PassthrougSubject` 和 `CurrentValueSubject`。

`PassthroughSubject`:

- 无初始值或当前值存储
- 只转发接收到的值给订阅者
- 适用于：
  - 按钮点击等事件触发的场景
  - 不需要保存状态的场景
  - 一次性通知

`CurrentValueSubject`:

- 必须提供初始值
- 维护一个当前值
- 新订阅者立即收到当前值
- 可通过 `.value` 属性读写当前值
- 适用于：
  - 状态管理（如开关状态）
  - 需要缓存最新值的场景
  - 需要立即知道当前状态的场景


| 特性   | PassthroughSubject     | CurrentValueSubject       |
| ------ | ---------------------- | ------------------------- |
| 初始化 | 不需要初始值           | 必须提供初始值            |
| 值存储 | 不存储值               | 存储最新值                |
| 新订阅 | 只接收订阅后的新值     | 立即接收当前值            |
| 值访问 | 无法访问当前值         | 可通过 value 属性访问     |
| 设置值 | 只能通过 send() 发送值 | 可用 send() 或 value 属性 |

```swift
import Combine
import Foundation

// PassthroughSubject
let passthroughSubject = PassthroughSubject<String, Never>()

// curValueSubject
let curValueSubject = CurrentValueSubject<String, Never>("初始值")

// 订阅 PassthroughSubject
passthroughSubject
    .sink { value in
        print("PassthroughSubject 接收到: \(value)")
    }

// 发送俩值
passthroughSubject.send("第一个值")
passthroughSubject.send("第二个值")

// 订阅 curValueSubject
curValueSubject
    .sink { value in
        print("curValueSubject 接收到: \(value)")
    }

// 发送新值
curValueSubject.send("更新后的值")

//输出当前值
print("curValueSubject 当前值: \(curValueSubject.value)")

// 发送另一个新值
curValueSubject.send("再次更新的值")

// 打印当前值
print("curValueSubject 当前值: \(curValueSubject.value)")

/* 
输出
PassthroughSubject 接收到: 第一个值
PassthroughSubject 接收到: 第二个值
curValueSubject 接收到: 初始值
curValueSubject 接收到: 更新后的值
curValueSubject 当前值: 更新后的值
curValueSubject 接收到: 再次更新的值
curValueSubject 当前值: 再次更新的值
*/
```

## 结语

Publisher 是 Combine 框架的基础组件之一，它为数据流的生成和传递提供了灵活而强大的工具。通过学习 Publisher 的工作原理和使用方式，开发者可以更有效地管理应用中的数据流。在下一篇文章中，将继续探讨介绍 Combine 中的订阅者（Subscriber）机制，进一步完善 Combine 知识体系。