# Swift Combine 学习（五）：Backpressure和 Scheduler

[TOC]

## 引言

在前面的文章中，已经介绍了 Combine 的基础概念、订阅机制和操作符的使用。本文将深入探讨 Combine 中的异步流程控制，包括 Backpressure 和 Scheduler。这些概念对于编写健壮的异步应用程序非常重要。

## 订阅的生命周期

Combine 中的订阅遵循以下生命周期：

1. 创建订阅：当调用 `Publisher` 的 `subscribe(_:)` 方法时，创建一个新的订阅。
2. 请求值：订阅者通过 `Subscription` 的 `request(_:)` 方法请求值。
3. 接收值：发布者通过调用订阅者的 `receive(_:)` 方法发送值。
4. 完成或取消：发布者通过两种方式终止：
   1. 调用 `receive(completion:)` 方法表示完成（成功或失败）
   2. 订阅者调用 `cancel()` 方法取消订阅。


## Backpressure

Combine 的 Backpressure 技术用于控制发布者（Publisher）向订阅者（Subscriber）发送数据的速率，防止订阅者因处理能力不足而被过多的数据淹没掉。Backpressure 本质上就是一种流量控制机制，确保系统在高负载或高并发情况下仍然能正常工作。在 Combine 中，Backpressure 通过 `Subscribers.Demand` 来处理：

- `Subscribers.Demand` 允许订阅者精确控制它希望接收的元素数量。
- 订阅者可以请求有限数量的元素（如`.max(n)`）、无限元素（`.unlimited`），或者不请求任何元素（`.none`）。
- 发布者必须尊重这个需求，不发送超过请求数量的元素。

Demand 的源码：

```swift
@frozen public struct Demand : Equatable, Comparable, Hashable, Codable, CustomStringConvertible {
        /// A request for as many values as the publisher can produce.
        public static let unlimited: Subscribers.Demand

        /// A request for no elements from the publisher.
        ///
        /// This is equivalent to `Demand.max(0)`.
        public static let none: Subscribers.Demand

        /// Creates a demand for the given maximum number of elements.
        ///
        /// The publisher is free to send fewer than the requested maximum number of elements.
        ///
        /// - Parameter value: The maximum number of elements. Providing a negative value for this parameter results in a fatal error.
        @inlinable public static func max(_ value: Int) -> Subscribers.Demand
        ...
}
```

一个简单的例子：在5秒快速数据生成器可以生成50个值，但由于 Backpressure，实际上只处理了5个值。用 Backpressure 防止数据消费者被过多的数据淹没。

```swift
import Combine
import Foundation
import PlaygroundSupport

class FastDataProducer: Publisher {
    typealias Output = Int
    typealias Failure = Never
    
    private var current = 0
    private var timer: Timer?
    private var subscriber: AnySubscriber<Int, Never>?
    
    private class FastDataSubscription: Subscription {
        private var producer: FastDataProducer?
        
        init(producer: FastDataProducer) {
            self.producer = producer
        }
        
        func request(_ demand: Subscribers.Demand) {
            // 写需求
        }
        
        func cancel() {
            producer?.timer?.invalidate()
            producer?.timer = nil
            producer?.subscriber = nil
            producer = nil
        }
    }
    
    func receive<S>(subscriber: S) where S : Subscriber, Failure == S.Failure, Output == S.Input {
        self.subscriber = AnySubscriber(subscriber)
        subscriber.receive(subscription: FastDataSubscription(producer: self))
        
        timer = Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) { [weak self] _ in
            guard let self = self else { return }
            _ = self.subscriber?.receive(self.current)
            self.current += 1
        }
    }
    
    func stop() {
        timer?.invalidate()
        timer = nil
        subscriber?.receive(completion: .finished)
        subscriber = nil
    }
}

class SlowDataConsumer: Subscriber {
    typealias Input = Int
    typealias Failure = Never
    
    private var subscription: Subscription?
    
    func receive(subscription: Subscription) {
        self.subscription = subscription
        subscription.request(.max(1))
    }
    
    func receive(_ input: Int) -> Subscribers.Demand {
        print("接收到值: \(input)")
        Thread.sleep(forTimeInterval: 1)
        return .max(1)
    }
    
    func receive(completion: Subscribers.Completion<Never>) {
        print("完成")
    }
}

let producer = FastDataProducer()
let consumer = SlowDataConsumer()
producer.subscribe(consumer)

DispatchQueue.main.asyncAfter(deadline: .now() + 5) {
    print("停止")
    producer.stop()
}

// 我是用 playground 运行，就使用 PlaygroundPage 防止过早退出
PlaygroundPage.current.needsIndefiniteExecution = true

DispatchQueue.main.asyncAfter(deadline: .now() + 6) {
    PlaygroundPage.current.finishExecution()
}


/*输出：
接收到值: 0
接收到值: 1
接收到值: 2
接收到值: 3
接收到值: 4
停止
完成
*/
```

## Scheduler

在 Swift Combine 框架中， Secheduler 是一个协议，它定义了如何调度工作。 Seheduler 可以被用来控制事件的执行时机和执行线程。

```swift
public protocol Scheduler<SchedulerTimeType> {

    /// Describes an instant in time for this scheduler.
    associatedtype SchedulerTimeType : Strideable where Self.SchedulerTimeType.Stride : SchedulerTimeIntervalConvertible

    /// A type that defines options accepted by the scheduler.
    ///
    /// This type is freely definable by each `Scheduler`. Typically, operations that take a `Scheduler` parameter will also take `SchedulerOptions`.
    associatedtype SchedulerOptions

    /// This scheduler’s definition of the current moment in time.
    var now: Self.SchedulerTimeType { get }

    /// The minimum tolerance allowed by the scheduler.
    /// 调度器容许的最小容差
    var minimumTolerance: Self.SchedulerTimeType.Stride { get }

    /// Performs the action at the next possible opportunity.
    /// 在下一个可能的时机执行
    func schedule(options: Self.SchedulerOptions?, _ action: @escaping () -> Void)

    /// Performs the action at some time after the specified date.
    /// 在指定日期之后的某个时间执行操作
    func schedule(after date: Self.SchedulerTimeType, tolerance: Self.SchedulerTimeType.Stride, options: Self.SchedulerOptions?, _ action: @escaping () -> Void)

    /// Performs the action at some time after the specified date, at the specified frequency, optionally taking into account tolerance if possible. 
    //// 在指定日期后，以给定频率执行。如果可能的话，可选择考虑公差。
    func schedule(after date: Self.SchedulerTimeType, interval: Self.SchedulerTimeType.Stride, tolerance: Self.SchedulerTimeType.Stride, options: Self.SchedulerOptions?, _ action: @escaping () -> Void) -> any Cancellable
}
```

三个主要方法：

1. `schedule(options: _:)`：安排一个工作项在调度器上执行。
2. `schedule(after:options: _:)`：安排一个工作项在指定时间后执行。
3. `schedule(after:interval:options: _:)`：安排一个工作项在指定时间后，以指定的间隔重复执行。

常见的 Seheduler 类型

* DispatchQueueScheduler：基于 `DispatchQueue` 的调度器，能够在指定的串行或并行队列上调度任务。适用于在主线程或后台线程上执行异步操作。

* RunLoopScheduler：基于 `RunLoop` 的调度器，适用于需要与用户界面事件交互的场景。它可以在主线程的事件循环中调度任务，通常用于处理与用户交互相关的操作。

* ImmediateScheduler：立即执行调度器，用于在当前调用堆栈中立即执行操作。这对于调试和测试非常有用，因为它允许在同步上下文中执行代码。

* TimerScheduler：基于 `Timer` 的调度器，用于定时执行任务。通常用于周期性任务，比如定时更新或轮询。

* CurrentValueSubjectScheduler：与 `CurrentValueSubject` 一起调度的调度器，允许在流中引入时间因素。

```swift
import Foundation
import Combine

final class SchedulerExample {
    private var cancellables: Set<AnyCancellable> = []
    private let backgroundQueue = DispatchQueue(label: "com.example.background")
    
    // MARK: - 基础调度方法
    private func basicSchedulerExample() -> Future<Void, Never> {
        Future { promise in
            print("\n🍎DispatchQueue Scheduler 示例")
            
            // 1. 主队列立即调度
            DispatchQueue.main.schedule {
                print("1️⃣ 主队列立即执行")
                print("   线程: \(Thread.current)")
            }
            
            // 2. 后台队列延迟调度
            let time = DispatchQueue.SchedulerTimeType(.now() + 1.0)
            self.backgroundQueue.schedule(after: time) {
                print("\n2️⃣ 后台队列延迟执行")
                print("   线程: \(Thread.current)")
            }
            
            // 3. 使用 Timer 进行周期性调度
            print("3️⃣ 开始周期性任务")
            Timer.publish(every: 1.0, on: .main, in: .common)
                .autoconnect()
                .prefix(2)  // 限制发出的元素数量为 2
                .sink { date in
                    print("   周期性任务触发: \(date)")
                    print("   线程: \(Thread.current)")
                }
                .store(in: &self.cancellables)
            
            // 3秒后完成这个示例
            DispatchQueue.main.schedule(after: .init(.now() + 3)) {
                promise(.success(()))
            }
        }
    }
    
    // MARK: - RunLoop
    private func runLoopExample() -> Future<Void, Never> {
        Future { promise in
            print("\n🍎RunLoop Scheduler 示例")
            
            // 1. 立即执行
            RunLoop.main.schedule {
                print("1️⃣ RunLoop 立即执行")
                print("   线程: \(Thread.current)")
            }
            
            // 2. 延迟执行
            let time = RunLoop.SchedulerTimeType(.now + 1.0)
            RunLoop.main.schedule(after: time) {
                print("\n2️⃣ RunLoop 延迟执行")
                print("   线程: \(Thread.current)")
            }
            
            // 2秒后完成
            DispatchQueue.main.schedule(after: .init(.now() + 2)) {
                promise(.success(()))
            }
        }
    }
    
    // MARK: - ImmediateScheduler 示例
    private func immediateExample() {
        print("🍎ImmediateScheduler 示例")
        
        ImmediateScheduler.shared.schedule {
            print("➡️ 同步立即执行")
            print("   线程: \(Thread.current)")
        }
    }
    
    // MARK: - CurrentValueSubject
    private func currentValueSubjectExample() -> Future<Void, Never> {
        Future { promise in
            print("\n🍎CurrentValueSubject 调度示例")
            
            let subject = CurrentValueSubject<Int, Never>(0)
            
            // 1. 在主队列上接收值
            subject
                .receive(on: DispatchQueue.main)
                .sink { value in
                    print("1️⃣ 主队列接收到值: \(value)")
                    print("   线程: \(Thread.current)")
                }
                .store(in: &self.cancellables)
            
            // 2. 在后台队列上接收值
            subject
                .receive(on: self.backgroundQueue)
                .sink { value in
                    print("2️⃣ 后台队列接收到值: \(value)")
                    print("   线程: \(Thread.current)")
                }
                .store(in: &self.cancellables)
            
            // 在不同时间发送值
            DispatchQueue.main.schedule(after: .init(.now() + 0.5)) {
                subject.send(1)
            }
            
            DispatchQueue.main.schedule(after: .init(.now() + 1.0)) {
                subject.send(2)
            }
            
            // 2秒后完成
            DispatchQueue.main.schedule(after: .init(.now() + 2)) {
                subject.send(completion: .finished)
                promise(.success(()))
            }
        }
    }
    
    func runAllExamples() {
        immediateExample()
        
        basicSchedulerExample()
            .flatMap { self.runLoopExample() }
            .flatMap { self.currentValueSubjectExample() }
            .sink { _ in
                print("\n✅ 所有示例执行完成")
            }
            .store(in: &cancellables)
    }
}

let ep = SchedulerExample()
ep.runAllExamples()

// 保持运行
RunLoop.main.run(until: Date(timeIntervalSinceNow: 6))

/*输出:
🍎ImmediateScheduler 示例
➡️ 同步立即执行
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}

🍎DispatchQueue Scheduler 示例
3️⃣ 开始周期性任务
1️⃣ 主队列立即执行
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}
   周期性任务触发: 2024-12-18 11:03:52 +0000
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}

2️⃣ 后台队列延迟执行
   线程: <NSThread: 0x600001729a00>{number = 9, name = (null)}
   周期性任务触发: 2024-12-18 11:03:53 +0000
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}

🍎RunLoop Scheduler 示例
1️⃣ RunLoop 立即执行
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}

2️⃣ RunLoop 延迟执行
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}

🍎CurrentValueSubject 调度示例
2️⃣ 后台队列接收到值: 0
   线程: <NSThread: 0x600001704580>{number = 5, name = (null)}
1️⃣ 主队列接收到值: 0
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}
1️⃣ 主队列接收到值: 1
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}
2️⃣ 后台队列接收到值: 1
   线程: <NSThread: 0x600001709b80>{number = 10, name = (null)}
1️⃣ 主队列接收到值: 2
   线程: <_NSMainThread: 0x600001708000>{number = 1, name = main}
2️⃣ 后台队列接收到值: 2
   线程: <NSThread: 0x600001729a00>{number = 9, name = (null)}

✅ 所有示例执行完成
*/
```

## 结语
Backpressure 和 Scheduler 是 Combine 框架中用于控制异步数据流的关键机制。通过掌握这些概念，开发者可以更好地管理数据流的速率和调度，提高应用的稳定性和性能。在下一篇文章中，我们将探索如何自定义 Publisher 和 Subscriber，以满足特定的应用需求。
