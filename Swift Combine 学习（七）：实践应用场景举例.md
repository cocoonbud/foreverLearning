# Swift Combine 学习（七）：实践应用场景举例

[TOC]

## 引言

在前面的系列文章中，我们已经深入学习了 Combine 框架的各个组成部分和使用方法。现在，是时候将这些理论知识付诸实践了。本文将通过实际的编程案例，展示 Combine 在日常开发中的应用场景，包括网络请求处理、用户界面交互、数据绑定等。通过这些实例，希望能够帮助您更好地理解如何在实际项目中使用 Combine。

## Combine 应用

### 网络请求处理

Combine 非常适合处理网络请求。以下是一个使用 Combine 进行网络请求的示例：

1. `NetworkService` 类定义了一个 `fetchPosts()` 方法，返回一个 `AnyPublisher<[Post], Error>`。
2. 使用 `URLSession.shared.dataTaskPublisher` 创建网络请求。
3. 使用 `map` 操作符提取响应数据。
4. 使用 `decode` 操作符将 JSON 数据解码为 `[Post]` 数组。
5. 使用 `@Published` 属性包装器声明 `posts` 数组,当它的值改变时会自动通知。
6. `fetchPosts()` 方法订阅网络请求的结果。

```swift
import UIKit
import Combine

enum NetworkError: Error {
    case invalidURL
    case requestFailed
    case decodingFailed
}

struct Post: Codable {
    let id: Int
    let title: String
    let body: String
}

class NetworkService {
    func fetchPosts() -> AnyPublisher<[Post], Error> {
        // 使用 JSONPlaceholder 提供的测试 API
        guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts") else {
            return Fail(error: NetworkError.invalidURL).eraseToAnyPublisher()
        }
        
        return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { output in
                guard let response = output.response as? HTTPURLResponse,
                      response.statusCode == 200 else {
                    throw NetworkError.requestFailed
                }
                return output.data
            }
            .decode(type: [Post].self, decoder: JSONDecoder())
            .mapError { error -> Error in
                switch error {
                case is URLError:
                    return NetworkError.requestFailed
                case is DecodingError:
                    return NetworkError.decodingFailed
                default:
                    return error
                }
            }
            .eraseToAnyPublisher()
    }
}

class PostsViewModel: ObservableObject {
    @Published var posts: [Post] = []
    private var cancellables = Set<AnyCancellable>()
    private let networkService = NetworkService()
    
    func fetchPosts() {
        print("   开始获取帖子...")
        
        networkService.fetchPosts()
            .receive(on: DispatchQueue.main)
            .sink { completion in
                switch completion {
                case .finished:
                    print("✅ 成功获取帖子")
                case .failure(let error):
                    print("❌ 获取帖子失败: \(error)")
                }
            } receiveValue: { [weak self] posts in
                self?.posts = posts
                print("📝 获取到 \(posts.count) 条帖子")
                // 打印前2条帖子的标题
                posts.prefix(2).forEach { post in
                    print("   标题: \(post.title)")
                }
            }
            .store(in: &cancellables)
    }
}

// test
func testNetworkRequest() {
    let viewModel = PostsViewModel()
    viewModel.fetchPosts()
    RunLoop.main.run(until: Date(timeIntervalSinceNow: 5))
}

print("🍎开始网络请求测试")
testNetworkRequest()

/*输出：
🍎 开始网络请求测试
   开始获取帖子...
📝 获取到 100 条帖子
   标题: sunt aut facere repellat provident occaecati excepturi optio reprehenderit
   标题: qui est esse
✅ 成功获取帖子
*/
```

### 请求链式调用

有时我们需要基于第一个请求的结果发起第二个请求：

```swift
import UIKit
import Combine
import Foundation
import SwiftUI

struct User: Codable {
    let id: Int
    let name: String
}

struct UserPosts: Codable {
    let user: User
    let posts: [Post]
}

enum NetworkError: Error {
    case invalidURL
    case requestFailed
    case decodingFailed
}

struct Post: Codable {
    let id: Int
    let title: String
    let body: String
}

class NetworkService {
    func fetchPosts() -> AnyPublisher<[Post], Error> {
        // 使用 JSONPlaceholder 提供的测试 API
        guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts") else {
            return Fail(error: NetworkError.invalidURL).eraseToAnyPublisher()
        }
        
        return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { output in
                guard let response = output.response as? HTTPURLResponse,
                      response.statusCode == 200 else {
                    throw NetworkError.requestFailed
                }
                return output.data
            }
            .decode(type: [Post].self, decoder: JSONDecoder())
            .mapError { error -> Error in
                switch error {
                case is URLError:
                    return NetworkError.requestFailed
                case is DecodingError:
                    return NetworkError.decodingFailed
                default:
                    return error
                }
            }
            .eraseToAnyPublisher()
    }
}

extension NetworkService {
    // fetch user info
    func fetchUser(id: Int) -> AnyPublisher<User, Error> {
        guard let url = URL(string: "https://jsonplaceholder.typicode.com/users/\(id)") else {
            return Fail(error: NetworkError.invalidURL).eraseToAnyPublisher()
        }
        
        return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { output in
                guard let response = output.response as? HTTPURLResponse,
                      response.statusCode == 200 else {
                    throw NetworkError.requestFailed
                }
                return output.data
            }
            .decode(type: User.self, decoder: JSONDecoder())
            .mapError { error -> Error in
                switch error {
                case is URLError:
                    return NetworkError.requestFailed
                case is DecodingError:
                    return NetworkError.decodingFailed
                default:
                    return error
                }
            }
            .eraseToAnyPublisher()
    }
    
    // fetch user post
    func fetchPosts(userID: Int) -> AnyPublisher<[Post], Error> {
        guard let url = URL(string: "https://jsonplaceholder.typicode.com/users/\(userID)/posts") else {
            return Fail(error: NetworkError.invalidURL).eraseToAnyPublisher()
        }
        
        return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { output in
                guard let response = output.response as? HTTPURLResponse,
                      response.statusCode == 200 else {
                    throw NetworkError.requestFailed
                }
                return output.data
            }
            .decode(type: [Post].self, decoder: JSONDecoder())
            .mapError { error -> Error in
                switch error {
                case is URLError:
                    return NetworkError.requestFailed
                case is DecodingError:
                    return NetworkError.decodingFailed
                default:
                    return error
                }
            }
            .eraseToAnyPublisher()
    }
    
    // 组合请求：fetch user info and posts
    func fetchUserAndPosts(userID: Int) -> AnyPublisher<UserPosts, Error> {
        fetchUser(id: userID)
            .flatMap { user -> AnyPublisher<UserPosts, Error> in
                self.fetchPosts(userID: user.id)
                    .map { posts in
                        UserPosts(user: user, posts: posts)
                    }
                    .eraseToAnyPublisher()
            }
            .eraseToAnyPublisher()
    }
}

// ViewModel
class UserPostsViewModel: ObservableObject {
    @Published var userPosts: UserPosts?
    private var cancellables = Set<AnyCancellable>()
    private let networkService = NetworkService()
    
    func fetchUserAndPosts(userID: Int) {
        print("   开始获取用户 \(userID) 的信息和帖子...")
        
        networkService.fetchUserAndPosts(userID: userID)
            .receive(on: DispatchQueue.main)
            .sink { completion in
                switch completion {
                case .finished:
                    print("✅ 成功获取用户信息和帖子")
                case .failure(let error):
                    print("❌ 获取失败: \(error)")
                }
            } receiveValue: { [weak self] userPosts in
                self?.userPosts = userPosts
                print("\n📝 用户信息：")
                print("   ID: \(userPosts.user.id)")
                print("   名字: \(userPosts.user.name)")
                print("\n📝 该用户的帖子（前2条）：")
                userPosts.posts.prefix(2).forEach { post in
                    print("   标题: \(post.title)")
                }
            }
            .store(in: &cancellables)
    }
}

print("🍎开始组合请求测试")
testUserAndPosts()

/*输出
🍎开始组合请求测试
  开始获取用户 1 的信息和帖子...

📝 用户信息：
   ID: 1
   名字: Leanne Graham

📝 该用户的帖子（前2条）：
   标题: sunt aut facere repellat provident occaecati excepturi optio reprehenderit
   标题: qui est esse
✅ 成功获取用户信息和帖子
*/
```

### 并发请求

当需要同时发起多个请求并等待所有结果时：

```swift
func fetchMultipleUsers(ids: [Int]) -> AnyPublisher<[User], Error> {
    let publishers = ids.map { fetchUser(id: $0) }
    
    return Publishers.MergeMany(publishers)
        .collect()
        .eraseToAnyPublisher()
}

func fetchUserAndProfile(userID: Int) -> AnyPublisher<(User, Profile), Error> {
  // 使用 zip 保顺序
    Publishers.Zip(
        fetchUser(id: userID),
        fetchProfile(userID: userID)
    )
    .eraseToAnyPublisher()
}
```

### 用户界面更新

Combine 还可以用于响应式地更新用户界面。以下是一个简单的示例，展示如何使用 Combine 更新 UIKit 界面:

1. 创建一个 UISearchBar 扩展，增加一个 `textDidChangePublisher` 属性。

```swift
extension UISearchBar {
    var textDidChangePublisher: AnyPublisher<String, Never> {
        NotificationCenter.default.publisher(for: UISearchBar.textDidChangeNotification, object: self)
            .compactMap { $0.object as? UISearchBar }
            .map { $0.text ?? "" }
            .eraseToAnyPublisher()
    }
}

class SearchViewModel {
    @Published private(set) var searchResults: [String] = []
    
    func search(query: String) {
        // 模拟网络请求
        DispatchQueue.global().asyncAfter(deadline: .now() + 0.5) {
            let results = (0..<10).map { "\(query) result \($0)" }
            self.searchResults = results
        }
    }
}
```

2. `setupBindings()` 方法设置了两个主要的数据流： 

   1. 搜索栏文本变化到搜索操作。
   2. 搜索结果到 UI 更新。

   ```swift
   class SearchViewController: UIViewController {
       // ... 声明一个 searchBar 、一个 tableview
       
       private var viewModel = SearchViewModel()
       private var cancellables = Set<AnyCancellable>()
       
       override func viewDidLoad() {
           super.viewDidLoad()
           setupBindings()
       }
       
       private func setupBindings() {
           searchBar.textDidChangePublisher
               .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
               .removeDuplicates()
               .sink { [weak self] searchText in
                   self?.viewModel.search(query: searchText)
               }
               .store(in: &cancellables)
           
           viewModel.$searchResults
               .receive(on: DispatchQueue.main)
               .sink { [weak self] _ in
                   self?.tableView.reloadData()
               }
               .store(in: &cancellables)
       }
   }
   ```

3. 使用 `@Published` 属性包装器声明 `searchResults`，允许外部订阅，但只允许内部修改。`search(query:)` 方法模拟一个异步网络请求。

   ```swift
   class SearchViewModel {
       @Published private(set) var searchResults: [String] = []
       
       func search(query: String) {
           DispatchQueue.global().asyncAfter(deadline: .now() + 0.5) {
               let results = (0..<10).map { "\(query) result \($0)" }
               self.searchResults = results
           }
       }
   }
   ```

### 数据绑定

Combine 非常适合用于数据绑定，特别是在 MVVM 架构中。以下是一个简单的例子:

1. `User` 类使用`@Published` 属性包装。
2. 使用 `Publishers.CombineLatest` 来响应任一属性的变化。
3. `UserVC` 订阅 ViewModel 的 `displayName` 属性，并更新 UI。

```swift
class User {
    @Published var name: String
    @Published var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

class UserViewModel {
    @Published private(set) var displayName: String = ""
    private var user: User
    private var cancellables = Set<AnyCancellable>()
    
    init(user: User) {
        self.user = user
        setupBindings()
    }
    
    private func setupBindings() {
        Publishers.CombineLatest($user.name, $user.age)
            .map { name, age in
                return "\(name) (\(age) years old)"
            }
            .assign(to: \.displayName, on: self)
            .store(in: &cancellables)
    }
}

class UserVC: UIViewController {
	  // ... 声明一个 nameLabel
    
    private var viewModel: UserViewModel!
    private var cancellables = Set<AnyCancellable>()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let user = User(name: "Joy", age: 91)
        viewModel = UserViewModel(user: user)
        
        viewModel.$displayName
            .receive(on: DispatchQueue.main)
            .sink { [weak self] displayName in
                self?.nameLabel.text = displayName
            }
            .store(in: &cancellables)
    }
}
```

## Combine VS RxSwift

[RxSwift](https://github.com/ReactiveX/RxSwift) 是 ReactiveX 中的一个。ReactiveX 还有 RxJava、RxJS、RxKotlin 等等。以下是一个简单的对照表：

| RxSwift    | Combine             | 说明                 |
| ---------- | ------------------- | -------------------- |
| Observable | Publisher           | 发送数据的源         |
| Observer   | Subscriber          | 接收数据的目标       |
| DisposeBag | Set<AnyCancellable> | 管理订阅生命周期     |
| Subject    | Subject             | 既可发送也可接收数据 |

以下是一些常用操作符的对照表：

| RxSwift              | Combine          | 说明                            |
| -------------------- | ---------------- | ------------------------------- |
| map                  | map              | 值转换                          |
| flatMap              | flatMap          | 转换为新的 Observable/Publisher |
| filter               | filter           | 过滤值                          |
| distinctUntilChanged | removeDuplicates | 去重                            |
| debounce             | debounce         | 防抖                            |
| throttle             | throttle         | 节流                            |
| merge                | merge            | 合并数据流                      |
| combineLatest        | combineLatest    | 组合最新值                      |
| zip                  | zip              | 配对组合                        |

虽然 Combine 和 RxSwift 都是响应式编程框架，但它们有很多不同之处。如:

1. 来源：

   * Combine 是 Apple 官方框架，内置于 Swift 和 iOS 13 及以上版本中。
   * RxSwift 是社区驱动的项目，适用于 iOS、macOS 和其他平台

2. 语法基本概念相似，具体的 API 和命名有所不同。

3. Combine 虽然内置了很多操作符，但还是比 Rxswift 少。Combine 和 RxSwift 的操作符对比 [RxSwift to Combine Cheatsheet](https://github.com/CombineCommunity/rxswift-to-combine-cheatsheet)

4. 功能集：

   - RxSwift 提供了更丰富的操作符，涵盖了更多场景。
   - Combine 的功能相对较少，但 API 设计更强调安全性和清晰的错误处理。

5. 平台支持：

   - Combine 仅支持 Apple 平台（iOS 13+、macOS 10.15+、tvOS 13+、watchOS 6+）。
   - RxSwift 支持更广泛的平台和 iOS 版本。

6. 学习曲线:

   - Combine 的学习曲线相对较缓，特别是对于已经熟悉 Swift 的开发者。
   - RxSwift 的学习曲线可能更陡峭，因为它包含了更多的概念和操作符。

7. 性能

   RxSwift 是纯 Swift 实现，Combine 实际使用性能会更优。

8. 版本支持

   Combine 支持的最低系统版本是 iOS 13。但是有开源的 [OpenCombine](https://github.com/OpenCombine/OpenCombine) 可以支持到 iOS 9。

以下是一个简单的对比示例:

```swift
例1：

// Combine
let combinePublisher = (1...5).publisher
    .map { $0 * 2 }
    .filter { $0 > 5 }
    .sink { value in
        print("Combine: \(value)")
    }

// RxSwift
let rxObservable = Observable.from(1...5)
    .map { $0 * 2 }
    .filter { $0 > 5 }
    .subscribe(onNext: { value in
        print("RxSwift: \(value)")
    })
```

```swift
例2
let disposeBag = DisposeBag()

let observable = Observable.of(1, 2, 3, 4, 5, 6)

// 用 RxSwift 的操作符进行过滤
observable
    .filter { $0 % 2 == 0 } // 只保留偶数
    .subscribe(onNext: { value in
        print("RxSwift - Even number: \(value)")
    })
    .disposed(by: disposeBag)

let publisher = [1, 2, 3, 4, 5, 6].publisher

// 用 Combine 的操作符进行过滤
let cancellable = publisher
    .filter { $0 % 2 == 0 } 
    .sink(receiveValue: { value in
        print("Combine - Even number: \(value)")
    })
```

虽然语法略有不同，基本概念和操作是相似的。

## 实践中注意点

在使用 Combine 时，以下是一些性能优化、实践、常见错误:

1. 合理使用调度器

   使用 `receive(on:)` 确保在正确的线程上执行操作。

   ```swift
   somePublisher
       .receive(on: DispatchQueue.main)
       .sink { ... }
   ```

2. 管理订阅生命周期

   始终存储和管理 `AnyCancellable` 对象，以确保在不再需要时取消订阅，防止内存泄漏，降低开销等。

   ```swift
   let publisher = [1, 2, 3, 4, 5].publisher
   
   var cancellable: AnyCancellable?
   
   func subscribeToPublisher() {
       cancellable = publisher
           .filter { $0 % 2 == 0 } 
           .sink(receiveValue: { value in
               print("Received value: \(value)")
           })
   }
   
   // 不再需要时，及时取消订阅
   func cancelSubscription() {
       cancellable?.cancel()
   }
   ```

3. 共享昂贵资源

   使用 `shareReplay` 来共享昂贵的操作结果。如网络请求或复杂计算，这样可以避免重复执行相同的操作，从而提高性能。

   ```swift
   let sharedPublisher = someExpensivePublisher
       .shareReplay(1)
       .eraseToAnyPublisher()
   ```

4. 优化加载体验

   使用 `prepend` 和 `append` 优化加载体验，以提供更流畅的用户体验。比如先显示缓存数据，然后更新为最新数据，减少用户的等待时间。

   ```swift
   let dataPublisher = loadDataFromCache()
       .append(loadDataFromNetwork())
   ```

5. 避免内存泄漏

   在闭包中使用 `[weak self]` 来避免循环引用。

   ```swift
   somePublisher
       .sink { [weak self] value in
           self?.balabala(value)
       }
       .store(in: &cancellables)
   ```

6. Future 和 Just 这俩 Publisher 在初始化完成后会立即执行闭包里的逻辑，这就可能会造成不符合预期的执行流程错误。

   ```Swift
   let IOHeavyTask = Future<Int, Never> { promise in
       print("开始耗时计算...")
       // 模拟耗时操作
       Thread.sleep(forTimeInterval: 2)
       let result = 42
       print("计算完成")
       promise(.success(result))
   }
   
   Thread.sleep(forTimeInterval: 1)
   
   print("准备订阅")
   let cancellable = IOHeavyTask.sink { value in
       print("收到结果: \(value)")
   }
   
   /*
   开始耗时计算...
   计算完成
   准备订阅
   收到结果: 42
   */
   ```
   
   在上面例子中，Future 在创建时就立即开始了耗时计算。在我们准备好订阅之前，计算就已经完成了。订阅者可能错过了计算过程，只能接收到最终结果。我们可以使用 Defferred，套在 Publisher 的外边，Deferred 允许延迟 Publisher 的创建，直到有订阅者订阅时才开始。
   
   ```Swift
   let IOHeavyTask = Deferred {
       Future<Int, Never> { promise in
           print("开始耗时计算...")
           // 模拟耗时操作
           Thread.sleep(forTimeInterval: 2)
           let result = 42
           print("计算完成")
           promise(.success(result))
       }
   }
   
   Thread.sleep(forTimeInterval: 1)
   
   print("准备订阅")
   let cancellable = IOHeavyTask.sink { value in
       print("收到结果: \(value)")
   }
   
   /*
   准备订阅
   开始耗时计算...
   计算完成
   收到结果: 42
   */
   ```
   
7. 有时候一些错误导致 Subscription 意外结束

   这里就不再举例了，简单说一下可以用 catch 来提供一个默认值或是替代的 Publisher 等等。

## 结语

简单说下关于 Combine 的个人愚见：

1. 实用性：Combine 确实提供了强大的工具来处理异步编程和事件流。建议逐步将 Combine 整合到项目中，从简单的用例开始，如网络请求处理或简单的 UI 更新。这样可以在实践中学习，同时避免在整个项目中过度使用导致的复杂性。
2. 性能 ：在大多数情况下，Combine 的性能表现良好。不过在处理大量高频事件时，得注意内存使用和 CPU 占用。使用诸如 `debounce` 或 `throttle` 等操作符可以有效控制事件流，提高应用性能。
3. 未来 ：随着 App 开发 iOS 最低兼容将要来到 iOS 13，SwiftUI 普及，Combine 在 iOS 开发中的重要性可能会随之进一步提升。需要持续关注  WWDC 和 Apple 的文档更新。

保持务实，根据项目需求和团队能力来决定使用的程度。
