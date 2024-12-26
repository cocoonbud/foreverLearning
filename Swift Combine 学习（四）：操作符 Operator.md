# Swift Combine 学习（四）：操作符 Operator

[TOC]

## 引言

在前几篇文章中，我们已经了解了 Combine 框架的基本概念、发布者和订阅者的工作机制。本文将详细介绍 Combine 中的操作符（Operator），这些操作符是处理和转换数据流的重要工具。通过学习各类操作符的使用，我们可以更灵活地处理异步事件流，构建复杂的数据处理链条，从而提升应用的响应能力和性能。

## 操作符 (`Operator`)

`Operator` 在 Combine 中用于处理、转换 `Publisher` 发出的数据。`Operator` 修改、过滤、组合或以其他方式操作数据流。Combine 提供了大量内置操作符，如：

* 转换操作符：如 `map`、`flatMap` 和 `scan`，用于改变数据的形式或结构。

  * `scan`：用于对上游发布者发出的值进行累加计算。它接收一个初始值和一个闭包，每次上游发布者发出一个新元素时，`scan` 会根据闭包计算新的累加值，并将累加结果传递给下游。

    ```swift
    let publisher = [1, 2, 3, 4].publisher
    publisher
        .scan(0, { a, b in
            a+b
        })
        .sink { print($0) }
    // 1 3 6 10
    ```

  * `map`：用于对上游发布者发出的值进行转换。它接收一个闭包，该闭包将每个从上游发布者接收到的值转换为新的值，然后将这个新值发给下游

    ```swift
    let nums = [1, 2, 3, 4, 5]
    let publisher = nums.publisher
    
    publisher
        .map { $0 * 10 }  // 将每个数乘以10
        .sink { 
            print($0)        
        }
    
    // 输出: 10 20 30 40 50
    ```

  * `flatMap`：用于将上游发布者发出的值转换为另一个发布者，并将新的发布者的值传递给下游。与 `map` 不同，它可以对发布者进行展平，消除嵌套。

    ```swift
    import Combine
    
    let publisher = [[1, 2, 3], [4, 5, 6]].publisher
    
    // 使用 flatMap 将每个数组转换为新的发布者并展平
    let cancellable = publisher
        .flatMap { arr in
            arr.publisher // 将每个数组转换为一个新的发布者
        }
        .sink { value in
            print(value)
        }
    
    /* 输出:
    1
    2
    3
    4
    5
    6
    */
    ```

* 过滤操作符：包括 `filter`、`compactMap` 和 `removeDuplicates`，用于选择性地处理某些数据。

  ```swift
  let numbers = ["1", "2", nil, "2", "4", "4", "5", "three", "6", "6", "6"]
  let publisher = numbers.publisher
  
  let subscription = publisher
      // 使用 compactMap 将字符串转换为整数。如果转换失败就过滤掉该元素
      .compactMap { $0.flatMap(Int.init) }
      // filter 过滤掉不符合条件的元素. 如过滤掉小于 3 的数
      .filter { $0 >= 3 }
      // 用 removeDuplicates 移除连续重复的元素
      .removeDuplicates()
      .sink {
          print($0)
      }
  
  // 输出: 4 5 6
  ```

* 组合操作符：如 `merge`、`zip` 和 `combineLatest`，用于将多个数据流合并成一个。

  * `combineLatest`：用于将多个发布者的最新值合成一个新的发布者。每当任何一个输入发布者发出新值时，`combineLatest` 操作符会将每个发布者的最新值组合并作为元组向下游发送。
  * `merge`：用于将多个发布者合并为一个单一的发布者，以不确定性的顺序发出所有输入发布者的值。
  * `zip`：用于将两个发布者组合成一个新的发布者，该发布者发出包含每个输入发布者的最新值的元组。

  ```swift
  let numberPublisher = ["1", "2", nil].publisher.compactMap { Int($0 ?? "") }
  let letterPublisher = ["A", "B", "C"].publisher
  let extraNumberPublisher = ["10", "20", "30"].publisher.compactMap { Int($0) }
  
  // 使用 merge 合并 numberPublisher 和 extraNumberPublisher
  print("Merge Example:")
  let mergeSubscription = numberPublisher
      .merge(with: extraNumberPublisher)
      .sink { value in
          print("Merge received: \(value)")
      }
  
  // 使用 zip 将 numberPublisher 和 letterPublisher 配对
  print("\n🍎Zip Example🍎")
  let zipSubscription = numberPublisher
      .zip(letterPublisher)
      .sink { number, letter in
          print("Zip received: number: \(number), letter: \(letter)")
      }
  
  // 使用 combineLatest 将 numberPublisher 和 letterPublisher 的最新值组合
  print("\n🍎CombineLatest Example🍎")
  let combineLatestSubscription = numberPublisher
      .combineLatest(letterPublisher)
      .sink { number, letter in
          print("CombineLatest received: number: \(number), letter: \(letter)")
      }
  
  /*输出
  Merge Example:
  Merge received: 1
  Merge received: 3
  Merge received: 10
  Merge received: 20
  Merge received: 30
  
  🍎Zip Example🍎
  Zip received: number: 1, letter: A
  Zip received: number: 3, letter: B
  
  🍎CombineLatest Example🍎
  CombineLatest received: number: 3, letter: A
  CombineLatest received: number: 3, letter: B
  CombineLatest received: number: 3, letter: C
  */
  ```

* 时间相关操作符：例如 `debounce`、`throttle` 和 `delay`，用于控制数据发送的时机。

  * `debounce`：在指定时间窗口内，如果没有新的事件到达，才会发布最后一个事件。通常用于防止过于频繁的触发，比如搜索框的实时搜索。
  * `throttle`：在指定时间间隔内，只发布一次。如果 `latest` 为 `true`，会发布时间段内的最后一个元素，`false` 时发布第一个元素。
  * `delay`：将事件的发布推迟指定时间。

  ```swift
  import UIKit
  import Combine
  import Foundation
  import SwiftUI
  
  class ViewController: UIViewController {
      var cancellableSets: Set<AnyCancellable>?
      
      override func viewDidLoad() {
          super.viewDidLoad()
          cancellableSets = Set<AnyCancellable>()
          
          testDebounce()
  //        testThrottle()
  //        testDelay()
      }
      
      func testDebounce() {
          print("🍎 Debounce Example 🍎")
          let searchText = PassthroughSubject<String, Never>()
          searchText
              .debounce(for: .seconds(0.3), scheduler: DispatchQueue.main)
              .sink { text in
                  print("Search request: \(text) at \(Date())")
              }.store(in: &cancellableSets!)
          
          // Simulate rapid input
          ["S", "Sw", "Swi", "Swif", "Swift"].enumerated().forEach { index, text in
              DispatchQueue.main.asyncAfter(deadline: .now() + Double(index) * 0.1) {
                  print("Input: \(text) at \(Date())")
                  searchText.send(text)
              }
          }
      }
      
      // Throttle Example
      func testThrottle() {
          print("🍎 Throttle Example 🍎")
          let scrollEvents = PassthroughSubject<Int, Never>()
          
          scrollEvents
              .throttle(for: .seconds(0.2), scheduler: DispatchQueue.main, latest: false)
              .sink { position in
                  print("Handle scroll position: \(position) at \(Date())")
              }
              .store(in: &cancellableSets!)
          
          // Simulate rapid scrolling
          (1...5).forEach { position in
              print("Scrolled to: \(position) at \(Date())")
              scrollEvents.send(position)
          }
      }
      
      // Delay Example
      func testDelay() {
          print("🍎 Delay Example 🍎")
          let notifications = PassthroughSubject<String, Never>()
          
          notifications
              .delay(for: .seconds(1), scheduler: DispatchQueue.main)
              .sink { message in
                  print("Display notification: \(message) at \(Date())")
              }
              .store(in: &cancellableSets!)
          
          print("Send notification: \(Date())")
          notifications.send("Operation completed")
      }
  }
  
  /*
  🍎 Debounce Example 🍎
  输入: S at 2024-10-21 09:23:19 +0000
  输入: Sw at 2024-10-21 09:23:19 +0000
  输入: Swi at 2024-10-21 09:23:19 +0000
  输入: Swif at 2024-10-21 09:23:19 +0000
  输入: Swift at 2024-10-21 09:23:19 +0000
  搜索请求: Swift at 2024-10-21 09:23:19 +0000
  */
  ```

* 错误处理操作符：如 `catch` 和 `retry`，用于处理错误情况。

* 处理多个订阅者：例如 `multicast` 和 `share`

  * `multicast`：使用 `multicast` 操作符时，它会将原始的 `Publisher` 包装成一个`ConnectablePublisher`，并且将所有订阅者的订阅合并为一个单一的订阅。这样，无论有多少个订阅者，原始的 `Publisher` 都只会收到一次 `receive(_:)` 调用，即对每个事件只处理一次。然后，`multicast` 操作符会将事件分发给所有的订阅者。

    ```swift
    import Combine
    
    var cancelables: Set<AnyCancellable> = Set<AnyCancellable>()
    
    let publisher = PassthroughSubject<Int, Never>()
    
    // 不使用 multicast() 的情况
    let randomPublisher1 = publisher
        .map { _ in Int.random(in: 1...100) }
    
    print("Without multicast():")
    randomPublisher1
        .sink {
            print("Subscriber 1 received: \($0)")
        }
        .store(in: &cancelables)
    
    randomPublisher1
        .sink {
            print("Subscriber 2 received: \($0)")
        }
        .store(in: &cancelables)
    
    publisher.send(1)
    
    let publisher2 = PassthroughSubject<Int, Never>()
    
    // 使用 multicast() 的情况
    let randomPublisher2 = publisher2
        .map { _ in Int.random(in: 1...100) }
        .multicast(subject: PassthroughSubject<Int, Never>())
    
    print("\nWith multicast():")
    randomPublisher2
        .sink {
            print("Subscriber 1 received: \($0)")
        }
        .store(in: &cancelables)
    
    randomPublisher2
        .sink {
            print("Subscriber 2 received: \($0)")
        }
        .store(in: &cancelables)
    
    let connect = randomPublisher2.connect()
    publisher2.send(1)
    
    /*输出:
    Without multicast():
    Subscriber 1 received: 43
    Subscriber 2 received: 39
    
    With multicast():
    Subscriber 1 received: 89
    Subscriber 2 received: 89
    */
    ```
    
  * `share`：它是一个自动连接的多播操作符，会在第一个订阅者订阅时开始发送值，并且会保持对上游发布者的订阅直到最后一个订阅者取消订阅。当多个订阅者订阅时，所有订阅者接收相同的输出，而不是每次订阅时重新触发数据流。
  
    ```swift
    import Combine
    
    var cancellables: Set<AnyCancellable> = Set<AnyCancellable>()
    
    let publisher = PassthroughSubject<Int, Never>()
    
    // 不使用 share() 的情况
    let randomPublisher1 = publisher
        .map { _ in Int.random(in: 1...100)
            
        }
    
    print("Without share():")
    randomPublisher1
        .sink {
            print("Subscriber 1 received: \($0)")
        }
        .store(in: &cancellables)
                  
    randomPublisher1
        .sink {
            print("Subscriber 2 received: \($0)")
        }
        .store(in: &cancellables)
    
    publisher.send(1)
    
    let publisher2 = PassthroughSubject<Int, Never>()
    
    // 使用 share() 的情况
    let randomPublisher2 = publisher2
        .map { _ in Int.random(in: 1...100)
            
        }
        .share()
    
    print("\nWith share():")
    randomPublisher2
        .sink {
            print("Subscriber 1 received: \($0)")
        }
        .store(in: &cancellables)
    
    randomPublisher2
        .sink {
            print("Subscriber 2 received: \($0)")
        }
        .store(in: &cancellables)
    
    publisher2.send(1)
    
    /*
    输出
    Without share():
    Subscriber 2 received: 61
    Subscriber 1 received: 62
    
    With share():
    Subscriber 2 received: 92
    Subscriber 1 received: 92
    */
    ```
  
  `share` 和 `multicast` 的区别：
  
  * 自动连接：使用 `share` 时，原始 `Publisher` 会在第一个订阅者订阅时自动连接，并在最后一个订阅者取消订阅时自动断开连接。
  * 无需手动连接：无需显式调用 `connect()` 方法来启动数据流，`share` 会自动管理连接。

我们可以使用这些操作符创建成一个链条。`Operator` 通常作为 `Publisher` 的扩展方法实现。

以下是一个简化的 `map` 操作符示例：

```swift
extension Publishers {
    struct Map<Upstream: Publisher, Output>: Publisher {
        typealias Failure = Upstream.Failure
        let upstream: Upstream
        let transform: (Upstream.Output) -> Output
        
        func receive<S: Subscriber>(subscriber: S) where S.Input == Output, S.Failure == Failure {
            upstream.subscribe(Subscriber(downstream: subscriber, transform: transform))
        }
    }
}

extension Publisher {
    func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.Map<Self, T> {
        return Publishers.Map(upstream: self, transform: transform)
    }
}
```

## 类型擦除（Type Erasure）

类型擦除（type erasure）允许在不暴露具体类型的情况下，对遵循相同协议的多个类型进行统一处理。换句话说，类型擦除可以将不同类型的数据包装成一个统一的类型，从而实现更灵活、清晰、通用的编程。

```swift
let publisher = Just(5)
    .map { $0 * 2 }
    .filter { $0 > 5 }
```

在这个简单的例子中 Publisher 的实际类型是 `Publishers.Filter<Publishers.Map<Just<Int>, Int>, Int>`。类型会变得非常复杂，特别是在使用多个操作符连接多个 `Publisher` 的时候。回到 Combine 中的 `AnySubscriber` 和 AnyPublisher，每个 `Publisher` 都有一个方法 `eraseToAnyPublisher()`，它可以返回一个 `AnyPublisher` 实例。就会被简化为 `AnyPublisher<Int, Never>`。

```Swift
let publisher: AnyPublisher<Int, Never> = Just(5)
    .map { $0 * 2 }
    .filter { $0 > 5 }
    .eraseToAnyPublisher()  // 使用 eraseToAnyPublisher 方法对 Publisher 进行类型擦除
```

因为是 Combine 的学习，在此不对类型擦除展开过多。

## 结语

操作符是 Combine 框架中强大的工具，它们使得数据流的处理和转换变得更加灵活和高效。通过掌握操作符的使用，开发者可以创建更复杂和功能强大的数据处理逻辑。在下一篇文章中，我们将深入探讨 Combine 中的 Backpressure 和 Scheduler，进一步提升对异步数据流的理解和控制调度能力。