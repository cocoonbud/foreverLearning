# Swift Combine 学习（六）：自定义 Publisher 和 Subscriber

[TOC]

## 引言

在前面的文章中，我们已经学习了 Combine 框架的核心概念和基础组件。本文将探讨如何自定义 Publisher 和 Subscriber，以满足特定的应用需求。通过自定义这些组件，开发者可以创建更加灵活和强大的数据流处理逻辑，适应不同的应用场景。

## 错误处理和重试机制

Combine 提供了多种处理错误和实现重试机制方法。以下是一些常用的错误处理操作符：

1. 使用 `tryMap` 进行错误检查和抛出。
2. 使用 `retry` 操作符在失败时进行重试。
3. 使用 `catch` 操作符处理错误并提供 fallback 值。

```swift
import Combine
import Foundation

enum ErrorType: Error {
    case numberTooLarge
}

var cancellables = Set<AnyCancellable>()

let numbers = [2, 5, 11, 99]
    .publisher
    .tryMap { number -> Int in
        // 检查数字是否大于10
        guard number <= 10 else {
            throw ErrorType.numberTooLarge
        }
        return number * 2
    }
    .retry(1)
    .catch { error -> AnyPublisher<Int, Never> in
        // 出错就默认返回 0
        print("❌ 错误: \(error)")
        return Just(0)
            .eraseToAnyPublisher()
    }
    .eraseToAnyPublisher()

print("🍎 开始处理数字")
numbers
    .sink { number in
        print("📍 结果: \(number)")
    }
    .store(in: &cancellables)

/*输出：
🍎 开始处理数字
📍 结果: 4
📍 结果: 10
📍 结果: 4
📍 结果: 10
❌ 错误: numberTooLarge
📍 结果: 0
*/
```

## 调试 Combine 代码

响应式编程因为传统的堆栈跟踪信息不足、异步执行和线程切换、一些操作符的链式调用可能使得代码逻辑比较抽象等等原因导致其一大痛点就是出现 bug 不好排查。Swift Combine 提供了几个有用的操作符来帮助调试：

1. 使用 `print` 打印所有事件。
2. 使用 `breakpoint` 在特定条件下触发断点。
3. 使用 `handleEvents` 在发布者生命周期的各个阶段插入自定义的调试代码。

```swift
import Combine

class CombineDebugDemo {
    private var cancellables = Set<AnyCancellable>()
    
    // MARK: - 使用 print 操作符追踪数据流
    func basicDebugDemo() {
        let numbers = (6...7).publisher
        
        let printDemo = numbers
            .print("🔍 数据流追踪")
        
        printDemo
            .sink { print("Print演示完成: \($0)") }
            receiveValue: { print("Print值: \($0)") }
            .store(in: &cancellables)
    }
    
    // MARK: - breakpoint 条件断点
    func breakpointDemo() {
        let numbers = (7...9).publisher
        
        let breakpointDemo = numbers
            .breakpoint(
                receiveOutput: { value in
                    let shouldBreak = value > 10
                    print("⚡️ 断点检查: 值 = \(value), 是否触发 = \(shouldBreak)")
                    return shouldBreak
                }
            )
        
        breakpointDemo
            .sink { print("断点演示完成: \($0)") }
            receiveValue: { print("断点值: \($0)") }
            .store(in: &cancellables)
    }
    
    // MARK: - 使用 handleEvents 监控完整生命周期
    func handleEventsDemo() {
        let numbers = (5...7).publisher
        
        let handleEventsDemo = numbers
            .handleEvents(
                receiveSubscription: { subscription in
                    print("🌟订阅开始: \(subscription)")
                },
                receiveOutput: { value in
                    print("⌛️准备发送值: \(value)")
                },
                receiveCompletion: { completion in
                    print("✅发送完成: \(completion)")
                },
                receiveCancel: {
                    print("❌订阅取消")
                },
                receiveRequest: { demand in
                    print("📧收到需求: \(demand)")
                }
            )
        
        handleEventsDemo
            .sink { print("事件处理演示完成: \($0)") }
            receiveValue: { print("事件处理值: \($0)") }
            .store(in: &cancellables)
    }
    
    // MARK: - 综合例子展示
    func comprehensiveDebugDemo() {
        let numbers = (16...18).publisher
        
        numbers
            // 1. 原始数据
            .print("\n1️⃣ 原始数据")
            // 2. 添加条件断点（可选）
            .breakpoint(receiveOutput: { value in
                let shouldBreak = value > 16
                print("2️⃣ 断点检查: 值 = \(value), 触发 = \(shouldBreak)")
                return shouldBreak
            })
            
            // 3. 完整的生命周期输出
            .handleEvents(
                receiveSubscription: { _ in print("3️⃣ 订阅开始") },
                receiveOutput: { print("3️⃣ 输出值: \($0)") },
                receiveCompletion: { print("3️⃣ 完成: \($0)") },
                receiveCancel: { print("3️⃣ 取消") },
                receiveRequest: { print("3️⃣ 需求: \($0)") }
            )
            .sink(
                receiveCompletion: { print("4️⃣ 最终完成: \($0)") },
                receiveValue: { print("4️⃣ 最终值: \($0)") }
            )
            .store(in: &cancellables)
    }
}

let demo = CombineDebugDemo()

print("\n🍎基础 print")
demo.basicDebugDemo()

print("\n🍎断点演示")
demo.breakpointDemo()

print("\n🍎事件处理演示")
demo.handleEventsDemo()

print("\n🍎综合调试演示")
demo.comprehensiveDebugDemo()

/*输出

🍎基础 print
🔍 数据流追踪: receive subscription: (6...7)
🔍 数据流追踪: request unlimited
🔍 数据流追踪: receive value: (6)
Print值: 6
🔍 数据流追踪: receive value: (7)
Print值: 7
🔍 数据流追踪: receive finished
Print演示完成: finished

🍎断点演示
⚡️ 断点检查: 值 = 7, 是否触发 = false
断点值: 7
⚡️ 断点检查: 值 = 8, 是否触发 = false
断点值: 8
⚡️ 断点检查: 值 = 9, 是否触发 = false
断点值: 9
断点演示完成: finished

🍎事件处理演示
🌟订阅开始: 5...7
📧收到需求: unlimited
⌛️准备发送值: 5
事件处理值: 5
⌛️准备发送值: 6
事件处理值: 6
⌛️准备发送值: 7
事件处理值: 7
✅发送完成: finished
事件处理演示完成: finished

🍎综合调试演示

1️⃣ 原始数据: receive subscription: (16...18)
3️⃣ 订阅开始
3️⃣ 需求: unlimited

1️⃣ 原始数据: request unlimited

1️⃣ 原始数据: receive value: (16)
2️⃣ 断点检查: 值 = 16, 触发 = false
3️⃣ 输出值: 16
4️⃣ 最终值: 16

1️⃣ 原始数据: receive value: (17)
2️⃣ 断点检查: 值 = 17, 触发 = true
3️⃣ 输出值: 17
4️⃣ 最终值: 17

1️⃣ 原始数据: receive value: (18)
2️⃣ 断点检查: 值 = 18, 触发 = true
*/
```

## 自定义 Publisher 和 Subscriber

iOS 大部分场景下开发者无需自定义 Publisher，因为有 KVO 、 Notification 等。不过有时可能需要创建自定义的 Publisher 或 Subscriber 来满足特定需求。比如封装已经有的异步 API 、有特殊的数据传递需求、实现一些当前 Combine 操作符无法满足的功能的时候。

* 创建一个自定义的 TimerPublisher。这个 TimerPublisher 将模拟一个计时器，每秒发布一个整数值。然后写一个自定义 TimerSubscriber 用于接收从 TimerPublisher 发布的值，并做相应的处理，例如在控制台中打印出接收到的值。

  ```swift
  import Combine
  import Foundation
  
  // 自定义 Publisher
  class TimerPublisher: Publisher, @unchecked Sendable {
      // 定义 Publisher 的输出类型和失败类型
      typealias Output = Int
      typealias Failure = Never
      
      private var counter = 0
      private var timer: Timer?
      // 使用 dictionary 存储多个订阅者及其需求
      private var subscribers: [UUID: (subscriber: AnySubscriber<Output, Failure>, demand: Subscribers.Demand)] = [:]
      
      deinit {
          stop()
      }
      
      // 接受 Subscriber 并建立连接
      func receive<S>(subscriber: S) where S : Subscriber, TimerPublisher.Failure == S.Failure, TimerPublisher.Output == S.Input {
          let id = UUID()
          // 创建一个 Subscription 并将其传递给 Subscriber
          let subscription = TimerSubscription(id: id, publisher: self)
          subscribers[id] = (AnySubscriber(subscriber), .none)
          subscriber.receive(subscription: subscription)
      }
      
      // 开始
      func start() {
          guard timer == nil else { return }
          timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
              self?.processValue()
          }
      }
      
      func stop() {
          timer?.invalidate()
          timer = nil
          // 发送完成信号给所有订阅者
          subscribers.values.forEach { $0.subscriber.receive(completion: .finished) }
          subscribers.removeAll()
      }
      
      // 处理订阅者的 demand
      fileprivate func updateDemand(for id: UUID, with newDemand: Subscribers.Demand) {
          if let subscriberInfo = subscribers[id] {
              subscribers[id] = (subscriberInfo.subscriber, subscriberInfo.demand + newDemand)
          }
      }
      
      // 取消特定订阅者
      fileprivate func cancelSubscription(for id: UUID) {
          subscribers.removeValue(forKey: id)
          if subscribers.isEmpty {
              stop()
          }
      }
      
      // 处理发送
      private func processValue() {
          counter += 1
          
          // 为每个有需求的订阅者发送值
          subscribers = subscribers.mapValues { subscriberInfo in
              var currentDemand = subscriberInfo.demand
              
              // 只在有需求时发送值
              if currentDemand > .none {
                  // 直接用 receive 返回的 Demand
                  let newDemand = subscriberInfo.subscriber.receive(counter)
                  currentDemand += newDemand
                  currentDemand -= 1
              }
              
              return (subscriberInfo.subscriber, currentDemand)
          }
      }
      
      // 自定义 Subscription
      private class TimerSubscription: Subscription {
          private var id: UUID
          private weak var publisher: TimerPublisher?
          
          init(id: UUID, publisher: TimerPublisher) {
              self.id = id
              self.publisher = publisher
          }
          
          // 处理 Subscriber 的请求
          func request(_ demand: Subscribers.Demand) {
              publisher?.updateDemand(for: id, with: demand)
              publisher?.start()
          }
          
          func cancel() {
              publisher?.cancelSubscription(for: id)
          }
      }
  }
  
  // 自定义 Subscriber
  class TimerSubscriber: Subscriber {
      let name: String
      
      init(name: String) {
          self.name = name
      }
      
      // 指定输入、失败类型
      typealias Input = Int
      typealias Failure = Never
      
      func receive(subscription: Subscription) {
          print("\(name): 订阅已开始")
          subscription.request(.max(3)) // 限制接收3个值
      }
      
      func receive(_ input: Int) -> Subscribers.Demand {
          print("\(name): 接收到的值: \(input)")
          return .none // 不请求更多的值
      }
      
      func receive(completion: Subscribers.Completion<Never>) {
          print("\(name): 订阅完成")
      }
  }
  
  let timerPublisher = TimerPublisher()
  
  // 创建多个 subscriber
  let subscriber1 = TimerSubscriber(name: "订阅者1")
  let subscriber2 = TimerSubscriber(name: "订阅者2")
  
  // 订阅
  timerPublisher.receive(subscriber: subscriber1)
  timerPublisher.receive(subscriber: subscriber2)
  
  // 5秒后停止发布
  DispatchQueue.main.asyncAfter(deadline: .now() + 5) {
      timerPublisher.stop()
  }
  
  /* 输出:
  订阅者1: 订阅已开始
  订阅者2: 订阅已开始
  订阅者1: 接收到的值: 1
  订阅者2: 接收到的值: 1
  订阅者1: 接收到的值: 2
  订阅者2: 接收到的值: 2
  订阅者1: 接收到的值: 3
  订阅者2: 接收到的值: 3
  订阅者1: 订阅完成
  订阅者2: 订阅完成
  */
  ```

## 结语

自定义 Publisher 和 Subscriber 为开发者提供了更大的灵活性，能够根据具体需求扩展 Combine 框架的功能。通过掌握自定义组件的技巧，开发者可以打造出更具适应性和扩展性的应用。在下一篇文章中，将通过实际案例来展示 Combine 的贴合日常开发的简化的应用场景，帮助更好地理解和应用。