记 xcode 16.1 beta 中 __mh_execute_header 编译报错

#### 问题描述

朋友升级了 xcode 16.1 beta。在 iOS 项目中接入 unity 时，其中有一步需要设置 ``[ufw setExecuteHeader: &_mh_execute_header];``

```objective-c
UnityFramework *UnityFrameworkLoad() {
    NSString *bundlePath = nil;
    bundlePath = [[NSBundle mainBundle] bundlePath];
    bundlePath = [bundlePath stringByAppendingString: @"/Frameworks/UnityFramework.framework"];
    
    NSBundle* bundle = [NSBundle bundleWithPath: bundlePath];
    if ([bundle isLoaded] == false) [bundle load];
    
    UnityFramework *ufw = [bundle.principalClass getInstance];
    if (![ufw appController]) {
      	// 首次初始化 Unity
        [ufw setExecuteHeader: &_mh_execute_header];
    }
    return ufw;
}
```

发现编译报错：

 **Undefined symbols for architecture arm64:**

 **"__mh_execute_header", referenced from:UnityFrameworkLoad() in PCAppDelegate.o**

**ld: symbol(s) not found for architecture arm64**

**clang++: error: linker command failed with exit code 1 (use -v to see invocation)** 

`__mh_execute_header` 它为程序提供了一个访问其自身 Mach-O 头部的方式。在 Unity 的集成中，这个符号可能被用来帮助 Unity 框架定位和初始化它需要的资源和代码段。

#### 解决

尝试写个 demo 发现在 xcode 15、16 beta 上却都能编译通过。

![image-20240827161101971](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//image-20240827161101971-20240827232641878.png)

最终在 [Xcode 16 beta release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes?changes=_1) 找到如下的一段话：**Some large or complex projects may fail to build and run if they are scanning for specific Mach-O sections in their binaries. (123416939)**。然后尝试把 Xcode 项目的 build settings 里的 `ENABLE_DEBUG_DYLIB` 设置为 NO，编译通过。

另外也有  [Apple Developer Forums](https://forums.developer.apple.com/forums/thread/760543) 类似问题的相关讨论。

#### 教训

1. 撑到最后正式版再升级。
2. 保持关注 xcode 发布说明。
3. 保持项目的可移植性，尽量减少此类使用方式。

