# Xcode WWDC2025 新东西

[toc]

## 优化

Xcode 下载大小降低，性能方面优化

* xcode 瘦身 24%，模拟器运行时默认不含 Intel 支持
* 优化了 xcode 输入的体验
* 加快 workspace 载入

## Workspace and editing

工作区和代码编辑器的改进

* 像 safari 浏览器一样的标签页

  * 可以整理成组或者是单一标签页

* 多词搜索

  * 以前搜索是按顺序，顺序不对就不匹配。现在比如输入 abc efg md，就会在项目中去搜包含这些临近词的所有组合，然后按照相关性进行排序。

    <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-10%20at%2017.39.02.png" alt="Screenshot 2025-06-10 at 17.39.02" style="zoom:50%;" />

* 语音写代码

  * 使用语音控制去写 swfit 代码。对于比如打球手手上、有困难的兄弟比较人性化。

* #Playground

  * 加了 `#Playground` 的部分，代码执行结果会自动显示在个新标签页面中。不用 run 起来代码打断点看变量、表达式的计算寄过，而且支持交互式的修改。如果是 swfitUI 只能看到运行的 UI 结果，用这个  `#Playground` 会比较方便的查看一些运行的比如属性是否符合预期。方便 debug ，提升效率。

  * 在 [swift 论坛](https://forums.swift.org/t/playground-macro-and-swift-play-idea-for-code-exploration-in-swift/79435) 上有老哥 22 年提出的这个 idea，现在被苹果实现加入到了 Xcode 中。amazing!!!

    <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-10%20at%2017.44.38.png" alt="Screenshot 2025-06-10 at 17.44.38" style="zoom:50%;" />

* icon composer

  * 可以了解下 [Say hello to the new look of app icons](Say hello to the new look of app icons)

* 字符串目录

  * 为本地化字符串加了类型安全的 swift symbols
  * 方便国际化
  * [Code-along: Explore localization with Xcode](https://developer.apple.com/videos/play/wwdc2025/225/)

## Intelligence

智能 AI coding

* 使用 chatGPT 等，实现类似 cursor、 Trae、 Windsurf 的功能。  
* intelligent 功能需要 xcode 26 配合 macOs26 才能使用。
* 我下载的 Xcode 26 安装后没有找到此功能入口， 升级了 MacOS26 后，系统切换 region 和使用美区 apple id，都还是显示区域不支持。
* 视频中有 debug，fixbug，但是修复的是一个很简单的问题。之后体验后再补充此部分。

[Writing code with intelligence in Xcode](https://developer.apple.com/documentation/xcode/writing-code-with-intelligence-in-xcode)

## Debugging and performance

新的调试和性能相关功能

* 调试 swift 并发代码时，xcode 能跟随执行流程深入到异步函数内部去 debug。

  * <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-10%20at%2018.42.44.png" alt="Screenshot 2025-06-10 at 18.42.44" style="zoom:50%;" />
  * <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot 2025-06-10 at 18.24.44.png" alt="Screenshot 2025-06-10 at 18.24.44" style="zoom:50%;" />
  * [What’s new in Swift](https://developer.apple.com/videos/play/wwdc2025/245/)

* 比如在调用相机权限的时候，没有写使用说明，之前就会 crash。现在 Xcode 会理解这个错误场景，会解释清楚，也增加有 Add 按钮直接跳转到 signing & capabilities 去修改，且会帮助修改更新 info plist 文件。

* **Instruments**

  * Processor Trace

    * 以前的 Instruments 是基于采样的分析器来了解 CPU 使用情况。比如下图采样点会错过橙色的运行部分。

      <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-10%20at%2018.35.59.png" alt="Screenshot 2025-06-10 at 18.35.59" style="zoom:50%;" />

    * M4 芯片和 iPhone 16 才行。Apple 芯片的最新设备可以扑捉处理器跟踪文件，里面存着的与 CPU 运行有关的信息，包含代码执行的分支和跳转指令。 现在可以连编译生成的代码，比如 swfit 的 ARC 内存管理都显示到执行流上。

  * CPU Counters

    * 此帮助理解代码运行 如何占用 CPU的工具大幅更新。
    * [Optimize CPU performance with Instruments](https://developer.apple.com/videos/play/wwdc2025/308/)

  * SwiftUI 性能

    * 优化 SwiftUI 性能，比如 Lists 更新的速度最快提升到之前的16 倍。
    * xcode 26 增加 SwiftUI 工具，帮助了解 SwiftUI 更新的原因和结果。
    * view body updates，帮助了解 App 中每个视图更新了多少次。
    * [What’s new in SwiftUI](https://developer.apple.com/videos/play/wwdc2025/256/)
    * [Optimize SwiftUI performance with Instruments](https://developer.apple.com/videos/play/wwdc2025/306/)

  * Power Profiler

    * 记录功耗指标，然后可视化呈现

    * Process 估计显示当前 App 对各种设备能源子组件比如 CPU 、 GPU 等影响。

      <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-10%20at%2019.04.30.png" alt="Screenshot 2025-06-10 at 19.04.30" style="zoom:50%;" />

    * 支持两种记录模式。1. 有线记录，将运行 Instruments 的设备与目标设备直接连接到一起。2. 通过设备的开发者设置，启动跟踪记录。

    * [Profile and optimize power usage in your app](https://developer.apple.com/videos/play/wwdc2025/226/)

* xcode organizer

  通过指标和诊断日志监控 App 的功耗和性能影响。

  * Trending Insights

     Xcode 16 官方加入了火焰图。Xcode 26 加入为挂起和启动诊断日志增加了 Trending Insights 功能。

    <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-10%20at%2019.15.13.png" alt="Screenshot 2025-06-10 at 19.15.13" style="zoom:50%;" />

  

## 构建 Building

关于构建方面的新动态

* xcode 16 推出显示构建（Explicit Modules）的模块，适用于 C 和 OC 代码。Xcode 26 显示构建的模块默认可以用于 Swift 代码。

* Explicit Modules for Swift，每个 Xcode 会把每个边一单元处理分为三个单独截断：1. scan 2. build modules 3.build  source

* 好处：1. 提升构建效率和可靠性 2. 提升 Swift 代码 debugging 的速度
* [Demystify explicitly built modules](https://developer.apple.com/videos/play/wwdc2024/10171/)
*  Swift Build
  * 是 Xcode、 Swift playground 使用的构建引擎，以及用于 Apple 自己操作系统的内部构建流程。Apple 也在努力将它整合到 Swift Package Manager 中，使得 Swfit 开源工具链和在 xcode 中的构建一致。
  * 开源了 [Swfit Build](https://github.com/swiftlang/swift-build)
  * [The Next Chapter in Swift Build Technologies](https://www.swift.org/blog/the-next-chapter-in-swift-build-technologies/)
* [Enhanced Security](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.hardened-process.enhanced-security-version/)
  * [Enabling enhanced security for your app](https://developer.apple.com/documentation/Xcode/enabling-enhanced-security-for-your-app)
  * 在 xcode 的 Signing & Capabilities 中增加 Enhanced Security。 启用后， xcode 会自动启用一些列安全相关的构建设置和权限。比如：
    * **指针认证（Pointer Authentication）**。防止篡改内存中的指针。启用后，系统会为应用创建的指针生成签名的元数据，然后在访问时验证签名是否被篡改过。要是签名无效，系统就抛出异常并 crash。
    * **类型分配器支持（Typed Allocator Support）**。 在 C 和 C++ 中，内存分配和管理容易出现不好追踪的错误，内存泄露、非法访问等等。有了这个Typed Allocator Support，Xcode 会启用 C 和 C++ 代码中的类型分配器。帮助咱开发编译时候检测潜在的内存问题。
    * **初始化栈变量为零（Initialize Stack Variables to Zero）**。xcode 会自动将变量初始化为 0，防止某些类型的 user-after-free 漏洞。防止会可能意外的使用到旧的值。
    * **安全相关的编译器警告（Security-Related Compiler Warnings）**。帮助识别潜在的不安全代码。
      * Eg：-Wsmpty-body：警告比如 for 循环或者是 if 语句这种控制流语句的主体是否为空。
    * **C++ 标准库加固和编译器边界检查（C++ Standard Library Hardening and Compiler Bounds Checking）**。可以防止数组越界等问题，xcode 会启动 fast mode，对标准库容器进行断言检查。
    * **额外运行时限制（Additional Runtime Platform Restrictions）**。 Mach 消息是重要的通信机制。对动态库加载和 Mach 消息进行额外检查，防止攻击者通过 Mach 端口获得提权漏洞等问题。
    * **只读内存用于内部平台状态（Read-Only Memory for Internal Platform State）**。将平台用于内部状态的内存区域标记为只读。如果应用修改了这些区域中的数据，系统会崩溃应用，提升系统安全性。

## 测试

一些测试体验的更新

* 增强 UI 自动化录制功能。将交互操作自动转化为测试代码。

  * <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//Screenshot%202025-06-11%20at%2002.04.34.png" alt="Screenshot 2025-06-11 at 02.04.34" style="zoom:50%;" />
  * <img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//image-20250611020718017.png" alt="image-20250611020718017" style="zoom:50%;" />
  * [Record, replay, and review: UI automation with Xcode](https://developer.apple.com/videos/play/wwdc2025/344/)

* 增加 XCTHitchMetric，帮助在 App 测试阶段发现卡顿。

  * 了解卡顿，可以看看之前的 tech talks，[Explore UI animation hitches and the render loop](https://developer.apple.com/videos/play/tech-talks/10855/)

* 推出更多运行时 API 检查

  * 进行运行时 API 检查防止性能下降。Runtime API checking 中有 Main Thread checker 、 Thread perfromance checker 、 other runtime issues

## 修复了一些问题（挑选）

* Xcode 在启动时恢复 UI 状态可能会崩溃。（152042209）
  * 从项目钟搜 UserInterfaceState.xcuserstate 文件，然后删掉。

* 升级 Xcode 后首次 build 时，模拟器可能无法启动。(152328794)
  * 等会重新 build 通常就 ok 了。（这也算是解决了？）
* 当选定的 Xcode 安装位置不在 /Applications/Xcode.app 时，打包工具（ba-package）和模拟服务会立即崩溃。（148927743）
  - 将想要的 Xcode 版本安装到 /Applications/Xcode.app，得使用 xcode-select 选中它。
* 已添加对 arm64e 架构支持的项目模拟器可能没法 build 成功。（123839235）
  * 从项目的架构中移除 arm64，或添加构建设置声明：`EXCLUDED_ARCHS[sdk=*simulator] = arm64e`
* 在 OC 中，对 nonatomic 属性的并发修改，现在可以产生有指导意义的 crash （148109501）
  * 合成的 setter 方法会短暂的存一个特殊哨兵值 `0x400000000000bad0`（32 位 watchOS 上是 0xbad0），就像放了个陷阱。这个值可能会被另一个不安全访问该属性的线程读到。那么如果在这个哨兵值上发生崩溃，就表明该属性的线程安全性有问题。帮助开发定位问题。



## 引用

[Xcode 26 Beta release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes)

[What’s new in Xcode](https://developer.apple.com/videos/play/wwdc2025/247/)













