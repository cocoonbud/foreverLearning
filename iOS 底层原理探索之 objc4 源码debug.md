#### 前言

你是否想调试 objc 源码，能断点跳跳跳跳进去，对 OC 底层一探究竟。于是你找到了各种官方开源源码，但是不能像我们日常 run 起来项目，进行调试。本文就手把手写清楚步骤，助你一臂之力。

**注意：如果你的 macOS 系统是 12，objc4-818.2 现在不支持，需要等 Apple 更新。另，文末有已可编译的 demo。**

#### 需要准备的资源

[objc4-818.2](https://opensource.apple.com/tarballs/objc4/)

[dyld-940](https://github.com/apple-oss-distributions/dyld/releases/tag/dyld-940)：The Dynamic Link Editor。Apple 的动态链接器。应用被编译打包成可执行文件之后，由 dyld 链接并加载程序。贯穿 App 启动过程，包含加载依赖库、主程序。

[dyld-852.2](https://github.com/apple-oss-distributions/dyld/releases/tag/dyld-852.2)

[Libc-1506.40.4](https://github.com/apple-oss-distributions/Libc/releases/tag/Libc-1506.40.4)

[libclosure-79](https://github.com/apple-oss-distributions/libclosure/releases/tag/libclosure-79)

[libplatform-273.40.1](https://github.com/apple-oss-distributions/libplatform/releases/tag/libplatform-273.40.1)

[libpthread-485.40.4](https://github.com/apple-oss-distributions/libpthread/releases/tag/libpthread-485.40.4)

[xnu-8019.41.5](https://github.com/apple-oss-distributions/xnu/releases/tag/xnu-8019.41.5)：XNU 内核是 Darwin 操作系统的一部分。用于 MacOS/iOS 系统。

[pthread_machdep.h](https://github.com/apple-oss-distributions/Libc/blob/Libc-825.40.1/pthreads/pthread_machdep.h)

[CrashReporterClient.h](https://github.com/apple-oss-distributions/Libc/blob/Libc-825.40.1/include/CrashReporterClient.h)

**注意：**以上资源都在不断更新

#### 解决编译遇到的问题

解压下载好的 objc4-818.2，然后打开 objc.xcodeproj。然后选择 objc 这个 target 编译一下。可以看到开始有报错。以下报错的先后次序不一定跟本文写的顺序完全一致。无非就是一些缺少引用文件，需要注释掉的代码。大家可以根据自己遇到的情况，在此文中搜索。

首先，在 objc4-818.2 工程的根目录下新建一个名为 THDependencies（文件名随意）的文件夹备用。

然后，在 Build Settings 中搜 Header Search Paths，设置下路径。

![image-20220209162058215](https://tva1.sinaimg.cn/large/008i3skNgy1gz7c2oyzlfj319107tdhm.jpg)

1. **’unable to find sdk 'macosx.internal'**

   分别设置 objc、objc-trampolines 这两个 target 的 BuildSettings 的 Base SDK 为 macOS。

2. **'sys/reason.h' file not found'**

   将之前下载好的资源中 xun 解压，然后在 **xnu/bsd/sys/** 目录下找到 **reason.h** 文件，copy 到 **THDependencies/sys/** （**THDependencies 中的 sys 为新创建，下同，缺啥补啥**）

3. **'mach-o/dyld_priv.h' file not found**

   在 **dyld/include/mach-o** 目录下找到 **dyld_priv.h**，copy 到 **THDependencies/mach-o/**

4. **dyld_priv.h 文件中，error: expected ',' extern dyld_platform_t dyld_get_base_platform(dyld_platform_t platform) __API_AVAILABLE(macos(10.14), ios(12.0), watchos(5.0), tvos(12.0), bridgeos(3.0)) **

   删除此文件中报错方法中的 **bridgeos(3.0)** 参数。

5. **'os/lock_private.h' file not found**

   在 **libplatform/private/os/** 找到 **lock_private.h**，copy 到 **THDependencies/os/**

6. **lock_private.h 中 error: expected ',' tvos(13.0), watchos(6.0), bridgeos(4.0)) = 0x00040000**

   删除 bridgeos(4.0) 这个参数。

7. **'pthread/tsd_private.h' file not found**

   在 **libpthread/private/pthread/** 目录下找到 **tsd_private.h**，copy 到 **THDependencies/pthread/**

8. **'os/base_private.h' file not found**

   在 **xun/libkern/os/** 找到 **base_private.h**，copy 到 **THDependencies/os/**

9. **'System/machine/cpu_capabilities.h' file not found**

   在 **xnu/osfmk/machine/** 目录下找到 **cpu_capabilities.h**, copy 到 **THDependencies/System/machine/**

10. **'os/tsd.h' file not found**

    在 **xnu/libsyscall/os/** 目录下找到 **tsd.h**, copy 到 **THDependencies/os/**

11. **'pthread/spinlock_private.h' file not found**

    在 **libpthread/private/pthread/** 目录下找到 **spinlock_private.h**，copy 到 **THDependencies/pthread/**

12. **'System/pthread_machdep.h' file not found**

    之前准备好的资源中 **pthread_machdep.h** copy 到 **THDependencies/System/**

13. **'pthread/spinlock_private.h' file not found**

    在 **libpthread/private/pthread/** 目录下找到 **spinlock_private.h**，copy 到 **THDependencies/pthread/**

14. **'os/feature_private.h' file not found**

    直接注释报错处，此头文件引用

15. **pthread_machdep.h 中的一些报错，**
    **如 Typedef redefinition with different types ('int' vs 'volatile OSSpinLock' (aka 'volatile int'))、**
    **Static declaration of '_pthread_has_direct_tsd' follows non-static declaration、_**
    **_Static declaration of '_pthread_getspecific_direct' follows non-static declaration、**
    **Static declaration of '_pthread_setspecific_direct' follows non-static declaration**

    注释掉报错处的代码、方法

16. **'CrashReporterClient.h' file not found**

    1. 之前准备好的资源中 **CrashReporterClient.h** copy 到 **THDependencies/**
    2. ![image-20220209180108648](https://tva1.sinaimg.cn/large/008i3skNgy1gz7eyxhkclj311g0ew76b.jpg)

17. **'objc-bp-assist.h' file not found**

    直接注释报错处，此头文件引用

18. **Use of undeclared identifier 'dyld_platform_version_macOS_10_13'**

    直接注释掉报错处，如下代码

    ```cpp
    if (!dyld_program_sdk_at_least(dyld_platform_version_macOS_10_13)) {
        DisableInitializeForkSafety = true;
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: disabling +initialize fork "
                         "safety enforcement because the app is "
                         "too old.)");
        }
    }
    ```

19. **'objc-shared-cache.h' file not found**

    在 **dyld/include/** 目录下找到 **objc-shared-cache.h**，copy 到 **THDependencies/**

20. **objc-opt.mm 文件中报 No member named 'objc_stringhash_offset_t' in namespace 'objc_opt' 等错误**

    下载个旧版本的 [**dyld-852.2**](https://github.com/apple-oss-distributions/dyld/releases/tag/dyld-852.2)，在 **dyld/include/** 目录下找到 **objc-shared-cache.h**，copy 到 **THDependencies/** 替换 **19** 中的 **objc-shared-cache.h** 文件。

21. **'os/linker_set.h' file not found** 

    在 **xnu/bsd/sys/** 目录下找到 **linker_set.h**，copy 到 **THDependencies/os/**

22. **'_simple.h' file not found**

    在 **libplatform/private/** 目录下找到 **_simple.h**，copy 到 **THDependencies/**

23. **'Block_private.h' file not found**

    在 **libclosure/** 目录下找到 **Block_private.h**，copy 到 **THDependencies/**

24. **'Cambria/Traps.h' file not found**

    直接注释报错处对此类的引用

25. **'kern/restartable.h' file not found**

    在 **xnu/osfmk/kern/** 目录下找到 **restartable.h**，copy 到 **THDependencies/kern/**

26. **'os/feature_private.h' file not found**

    直接注释报错处对此类的引用

27. **Use of undeclared identifier 'oah_is_current_process_translated'**

    ![image-20220210112412480](https://tva1.sinaimg.cn/large/008i3skNgy1gz8947i7lcj318c0asjt0.jpg)
    改为
    ![image-20220210112448103](https://tva1.sinaimg.cn/large/008i3skNgy1gz894skjuoj311308k3zd.jpg)

28. **'os/reason_private.h' file not found**

    在 **xnu/libkern/os/** 目录下找到 **reason_private.h**，copy 到 **THDependencies/os/**

29. **'os/variant_private.h' file not found**

    在 **Libc/os/** 目录下找到 **variant_private.h**，copy 到 **THDependencies/os/**

30. **Use of undeclared identifier 'dyld_fall_2020_os_versions'**

    ![image-20220210113044238](https://tva1.sinaimg.cn/large/008i3skNgy1gz89ayivkbj30mk07gq3y.jpg)
    改为
    ![image-20220210113117110](https://tva1.sinaimg.cn/large/008i3skNgy1gz89bj3n6lj30o107ht9m.jpg)

31. **objc-runtime.mm 文件中报 Use of undeclared identifier 'objc4'**

    ![image-20220210113425201](https://tva1.sinaimg.cn/large/008i3skNgy1gz89esojstj30q507rdgo.jpg)
    改为
    ![image-20220210113450266](https://tva1.sinaimg.cn/large/008i3skNgy1gz89f820zsj30qj07g3z8.jpg)

32. **Use of undeclared identifier 'dyld_platform_version_macOS_10_11'**

    ![image-20220210113605129](https://tva1.sinaimg.cn/large/008i3skNgy1gz89gj0y38j30or0bpdh8.jpg)
    改为
    ![image-20220210113629152](https://tva1.sinaimg.cn/large/008i3skNgy1gz89gxwlw3j30rd0blq4a.jpg)

33. **variant_private.h 文件中报 error: expected ',' API_AVAILABLE(macosx(10.16), ios(14.0), tvos(13.0), watchos(7.0), bridgeos(4.0)) 等**

    删除报错处 **bridgeos** 参数

34. **'_static_assert' declared as an array with a negative size**

    注释掉报错处的代码

    ```cpp
    //STATIC_ASSERT((~ISA_MASK & MACH_VM_MAX_ADDRESS) == 0  ||
    //              ISA_MASK + sizeof(void*) == MACH_VM_MAX_ADDRESS);
    ```

    

35. **Use of undeclared identifier 'dyld_platform_version_macOS_10_12'**

    注释掉报错处的 if 判断

36. **Use of undeclared identifier 'dyld_fall_2018_os_versions'**

    在 **objc-runtime-new.mm** 文件的报错处，注释掉 initializeTaggedPointerObfuscator 方法内的实现。

37. **Can't open order file: /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.1.sdk/AppleInternal/OrderFiles/libobjc.order**

    在 **objc** 这个 **target** 的 **Build Settings** 中，搜索 **Order File**，然后在其中添加搜索路径 **$(SRCROOT)/libobjc.order**

38. **Library not found for -lCrashReporterClient**

    在 **objc** 这个 **target** 的 **Build Settings** 中，搜索 **Other Linker Flags**，然后在其中删除 **-lCrashReporterClient**

39. **Library not found for -loah**

    在 **objc** 这个 **target** 的 **Build Settings** 中，搜索 **Other Linker Flags**，然后在其中删除 **-loah**

40. **error: SDK "macosx.internal" cannot be located.**

    ![image-20220210115516793](https://tva1.sinaimg.cn/large/008i3skNgy1gz8a0i8chdj313u0efmz7.jpg)

41. **unable to find sdk 'macosx.internal'**

    ![image-20240322234641585](/Users/spell/Library/Application Support/typora-user-images/image-20240322234641585.png)

    需要在 **PROJECT  objc** 的 **Build Settings** 中设置 **Base SDK** 为 **MacOS**。 **objc targe**t 的 **Build Settings** 中设置 **Base SDK** 为 **MacOS** 即可。

    ![image-20240322235034595](/Users/spell/Library/Application Support/typora-user-images/image-20240322235034595.png)

#### 调试举例

1. 新建一个 target。如 THObjcDebugDemo

2. 添加依赖。

   ![image-20220210143433247](https://tva1.sinaimg.cn/large/008i3skNgy1gz8em88e3cj30wm0ant9h.jpg)

3. 然后就可以自由发挥了。

注意：此时你在底层源码中打断点，会发觉断点无效。需要在 **Build Settings **中将 **Enable Hardened Runtime** 设为 **NO**。

#### [Objc4Debug](https://github.com/cocoonbud/Objc4Debug)

