对于 Objective-C 的一些实现，我们可以在 Apple 开源网站上下载 [objc4](https://opensource.apple.com/source/objc4/) 等源码一探究竟，之前也写了一篇如何 [debug objc4 源码的文章](https://juejin.cn/post/7064610907409612814)。这篇文章是从汇编角度简单的去窥探一下 Objective-C 的一些实现，个人记录下。如 Class Metada、属性、对成员变量的访问、调用类方法、调用实例方法、block 这几个基础常用点。

### 前言

Objective-C 源文件(.m) 的编译器是 Clang + LLVM，Swift 源文件的编译器是 swift + LLVM。借助 clang 命令，可以查看 .m 文件的汇编结果。

`xcrun --sdk iphoneos clang -arch arm64 -S XXXX.m`

`-arch <arm|arm64|x86_64|i386>`: 生成的代码的架构体系

`-S`：源代码文件。`-o`：输出文件。

生成汇编代码后，可以比照  [objc4](https://opensource.apple.com/source/objc4/) 中的源码对应去理解。

### Class Metadata

1.  `@interface`

   ```objective-c
   #import "THNormalClass.h"
   
   @interface THNormalClass()
   
   @end
   ```

   ```
   	.section	__TEXT,__text,regular,pure_instructions
   	.build_version ios, 15, 2	sdk_version 15, 2
   	.section	__DATA,__objc_imageinfo,regular,no_dead_strip
   L_OBJC_IMAGE_INFO:
   	.long	0
   	.long	64
   
   .subsections_via_symbol
   ```

   编译器对 `@interface` 并未产生啥汇编代码。

2. `@implementation`

   ```objective-c
   #import "THNormalClass.h"
   
   @interface THNormalClass()
   
   @end
   
   @implementation THNormalClass
   
   @end
   ```

   ```
   	.section	__TEXT,__text,regular,pure_instructions
   	.build_version ios, 15, 2	sdk_version 15, 2
   	.section	__TEXT,__objc_classname,cstring_literals
   l_OBJC_CLASS_NAME_:                     ; @OBJC_CLASS_NAME_
   	.asciz	"THNormalClass"
   
   	.section	__DATA,__objc_const
   	.p2align	3                               ; @"_OBJC_METACLASS_RO_$_THNormalClass"
   __OBJC_METACLASS_RO_$_THNormalClass:
   	.long	1                               ; 0x1
   	.long	40                              ; 0x28
   	.long	40                              ; 0x28
   	.space	4
   	.quad	0
   	.quad	l_OBJC_CLASS_NAME_
   	.quad	0
   	.quad	0
   	.quad	0
   	.quad	0
   	.quad	0
   
   	.section	__DATA,__objc_data
   	.globl	_OBJC_METACLASS_$_THNormalClass ; @"OBJC_METACLASS_$_THNormalClass"
   	.p2align	3
   _OBJC_METACLASS_$_THNormalClass:
   	.quad	_OBJC_METACLASS_$_NSObject
   	.quad	_OBJC_METACLASS_$_NSObject
   	.quad	__objc_empty_cache
   	.quad	0
   	.quad	__OBJC_METACLASS_RO_$_THNormalClass
   
   	.section	__DATA,__objc_const
   	.p2align	3                               ; @"_OBJC_CLASS_RO_$_THNormalClass"
   __OBJC_CLASS_RO_$_THNormalClass:
   	.long	0                               ; 0x0
   	.long	8                               ; 0x8
   	.long	8                               ; 0x8
   	.space	4
   	.quad	0
   	.quad	l_OBJC_CLASS_NAME_
   	.quad	0
   	.quad	0
   	.quad	0
   	.quad	0
   	.quad	0
   
   	.section	__DATA,__objc_data
   	.globl	_OBJC_CLASS_$_THNormalClass     ; @"OBJC_CLASS_$_THNormalClass"
   	.p2align	3
   _OBJC_CLASS_$_THNormalClass:
   	.quad	_OBJC_METACLASS_$_THNormalClass
   	.quad	_OBJC_CLASS_$_NSObject
   	.quad	__objc_empty_cache
   	.quad	0
   	.quad	__OBJC_CLASS_RO_$_THNormalClass
   
   	.section	__DATA,__objc_classlist,regular,no_dead_strip
   	.p2align	3                               ; @"OBJC_LABEL_CLASS_$"
   l_OBJC_LABEL_CLASS_$:
   	.quad	_OBJC_CLASS_$_THNormalClass
   
   	.section	__DATA,__objc_imageinfo,regular,no_dead_strip
   L_OBJC_IMAGE_INFO:
   	.long	0
   	.long	64
   
   .subsections_via_symbols
   ```

   - `__OBJC_CLASS_RO_$_THNormalClass` 和 `__OBJC_METACLASS_RO_$_THNormalClass` 对应 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-818.2/runtime/objc-runtime-new.h.auto.html) 中 `struct class_ro_t`

     ```c++
     struct class_ro_t {
             uint32_t flags;
             uint32_t instanceStart;
             uint32_t instanceSize;                    // instance 对象占用的内存控件
             const uint8_t * ivarLayout;
             const char * name;                        // 类名
             method_list_t * baseMethodList;           // 只读
             protocol_list_t * baseProtocols;
             const ivar_list_t * ivars;                // 成员变量列表
             const uint8_t * weakIvarLayout;
             property_list_t *baseProperties;
             method_list_t *baseMethods() const {
                 return baseMethodList;
             }
         };
     ```

   - `_OBJC_CLASS_$_THNormalClass` 和 `_OBJC_METACLASS_$_THNormalClass` 对应 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-818.2/runtime/objc-runtime-new.h.auto.html) 中 `struct objc_class`

     ```c++
     struct objc_class : objc_object {
             // Class ISA;
             Class superclass;
             cache_t cache;          // formerly cache pointer and vtable  方法缓存
             class_data_bits_t bits; // 用于获取具体的类信息
      
             class_rw_t *data() {
                 return bits.data(); //  class_rw_t* data() { return (class_rw_t *)(bits & FAST_DATA_MASK); }
             }
             .... // 其他的都是方法
         }
     ```

### 属性

```objective-c
@interface THNormalClass()

@property(nonatomic, copy) NSString *str1;

@property(nonatomic, copy) NSString *str2;

@end

@implementation THNormalClass

@end
```

```
	.section	__TEXT,__text,regular,pure_instructions
	.build_version ios, 15, 2	sdk_version 15, 2
	.p2align	2                               ; -- Begin function -[THNormalClass str1]
"-[THNormalClass str1]":                ; @"\01-[THNormalClass str1]"
...
"-[THNormalClass setStr1:]":            ; @"\01-[THNormalClass setStr1:]"
...
"-[THNormalClass str2]":                ; @"\01-[THNormalClass str2]"
...
"-[THNormalClass setStr2:]":            ; @"\01-[THNormalClass setStr2:]"
...
l_OBJC_CLASS_NAME_:                     ; @OBJC_CLASS_NAME_
...
__OBJC_METACLASS_RO_$_THNormalClass:
...
_OBJC_METACLASS_$_THNormalClass:
...
l_OBJC_METH_VAR_NAME_:                  ; @OBJC_METH_VAR_NAME_
	.asciz	"str1"

	.section	__TEXT,__objc_methtype,cstring_literals
l_OBJC_METH_VAR_TYPE_:                  ; @OBJC_METH_VAR_TYPE_
	.asciz	"@16@0:8"

	.section	__TEXT,__objc_methname,cstring_literals
l_OBJC_METH_VAR_NAME_.1:                ; @OBJC_METH_VAR_NAME_.1
	.asciz	"setStr1:"

...
__OBJC_$_INSTANCE_METHODS_THNormalClass:
	.long	24                              ; 0x18
	.long	4                               ; 0x4
	.quad	l_OBJC_METH_VAR_NAME_
	.quad	l_OBJC_METH_VAR_TYPE_
	.quad	"-[THNormalClass str1]"
	.quad	l_OBJC_METH_VAR_NAME_.1
	.quad	l_OBJC_METH_VAR_TYPE_.2
	.quad	"-[THNormalClass setStr1:]"
	.quad	l_OBJC_METH_VAR_NAME_.3
	.quad	l_OBJC_METH_VAR_TYPE_
	.quad	"-[THNormalClass str2]"
	.quad	l_OBJC_METH_VAR_NAME_.4
	.quad	l_OBJC_METH_VAR_TYPE_.2
	.quad	"-[THNormalClass setStr2:]"

	.private_extern	_OBJC_IVAR_$_THNormalClass._str1 ; @"OBJC_IVAR_$_THNormalClass._str1"
	.section	__DATA,__objc_ivar
	.globl	_OBJC_IVAR_$_THNormalClass._str1
	.p2align	2
_OBJC_IVAR_$_THNormalClass._str1:
	.long	8                               ; 0x8

	.section	__TEXT,__objc_methname,cstring_literals
l_OBJC_METH_VAR_NAME_.5:                ; @OBJC_METH_VAR_NAME_.5
	.asciz	"_str1"

	.section	__TEXT,__objc_methtype,cstring_literals
l_OBJC_METH_VAR_TYPE_.6:                ; @OBJC_METH_VAR_TYPE_.6
	.asciz	"@\"NSString\""

	.private_extern	_OBJC_IVAR_$_THNormalClass._str2 ; @"OBJC_IVAR_$_THNormalClass._str2"
	.section	__DATA,__objc_ivar
	.globl	_OBJC_IVAR_$_THNormalClass._str2
	.p2align	2
_OBJC_IVAR_$_THNormalClass._str2:
	.long	16                              ; 0x10

	.section	__TEXT,__objc_methname,cstring_literals
l_OBJC_METH_VAR_NAME_.7:                ; @OBJC_METH_VAR_NAME_.7
	.asciz	"_str2"

	.section	__DATA,__objc_const
	.p2align	3                               ; @"_OBJC_$_INSTANCE_VARIABLES_THNormalClass"
__OBJC_$_INSTANCE_VARIABLES_THNormalClass:
	.long	32                              ; 0x20
	.long	2                               ; 0x2
	.quad	_OBJC_IVAR_$_THNormalClass._str1
	.quad	l_OBJC_METH_VAR_NAME_.5
	.quad	l_OBJC_METH_VAR_TYPE_.6
	.long	3                               ; 0x3
	.long	8                               ; 0x8
	.quad	_OBJC_IVAR_$_THNormalClass._str2
	.quad	l_OBJC_METH_VAR_NAME_.7
	.quad	l_OBJC_METH_VAR_TYPE_.6
	.long	3                               ; 0x3
	.long	8                               ; 0x8

	.section	__TEXT,__objc_methname,cstring_literals
l_OBJC_PROP_NAME_ATTR_:                 ; @OBJC_PROP_NAME_ATTR_
	.asciz	"str1"

l_OBJC_PROP_NAME_ATTR_.8:               ; @OBJC_PROP_NAME_ATTR_.8
	.asciz	"T@\"NSString\",C,N,V_str1"

l_OBJC_PROP_NAME_ATTR_.9:               ; @OBJC_PROP_NAME_ATTR_.9
	.asciz	"str2"

l_OBJC_PROP_NAME_ATTR_.10:              ; @OBJC_PROP_NAME_ATTR_.10
	.asciz	"T@\"NSString\",C,N,V_str2"

	.section	__DATA,__objc_const
	.p2align	3                               ; @"_OBJC_$_PROP_LIST_THNormalClass"
__OBJC_$_PROP_LIST_THNormalClass:
	.long	16                              ; 0x10
	.long	2                               ; 0x2
	.quad	l_OBJC_PROP_NAME_ATTR_
	.quad	l_OBJC_PROP_NAME_ATTR_.8
	.quad	l_OBJC_PROP_NAME_ATTR_.9
	.quad	l_OBJC_PROP_NAME_ATTR_.10

	.p2align	3                               ; @"_OBJC_CLASS_RO_$_THNormalClass"
__OBJC_CLASS_RO_$_THNormalClass:
	.long	0                               ; 0x0
	.long	8                               ; 0x8
	.long	24                              ; 0x18
	.space	4
	.quad	0
	.quad	l_OBJC_CLASS_NAME_
	.quad	__OBJC_$_INSTANCE_METHODS_THNormalClass
	.quad	0
	.quad	__OBJC_$_INSTANCE_VARIABLES_THNormalClass
	.quad	0
	.quad	__OBJC_$_PROP_LIST_THNormalClass

	.section	__DATA,__objc_data
	.globl	_OBJC_CLASS_$_THNormalClass     ; @"OBJC_CLASS_$_THNormalClass"
	.p2align	3
_OBJC_CLASS_$_THNormalClass:
	.quad	_OBJC_METACLASS_$_THNormalClass
	.quad	_OBJC_CLASS_$_NSObject
	.quad	__objc_empty_cache
	.quad	0
	.quad	__OBJC_CLASS_RO_$_THNormalClass

	.section	__DATA,__objc_classlist,regular,no_dead_strip
	.p2align	3                               ; @"OBJC_LABEL_CLASS_$"
l_OBJC_LABEL_CLASS_$:
	.quad	_OBJC_CLASS_$_THNormalClass

	.section	__DATA,__objc_imageinfo,regular,no_dead_strip
L_OBJC_IMAGE_INFO:
	.long	0
	.long	64

.subsections_via_symbols
```

 `__OBJC_$_INSTANCE_VARIABLES_THNormalClass` 对应 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-818.2/runtime/objc-runtime-new.h.auto.html) 中的 `struct ivar_list_t` 加上 2 个`.long`（4 bytes）的开销。

```
__OBJC_$_INSTANCE_VARIABLES_THNormalClass:
	.long	32                              ; 0x20
	.long	2                               ; 0x2
	.quad	_OBJC_IVAR_$_THNormalClass._str1
	.quad	l_OBJC_METH_VAR_NAME_.5
	.quad	l_OBJC_METH_VAR_TYPE_.6
	.long	3                               ; 0x3
	.long	8                               ; 0x8
	.quad	_OBJC_IVAR_$_THNormalClass._str2
	.quad	l_OBJC_METH_VAR_NAME_.7
	.quad	l_OBJC_METH_VAR_TYPE_.6
	.long	3                               ; 0x3
	.long	8                               ; 0x8
```

对应 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-818.2/runtime/objc-runtime-new.h.auto.html) 中的 `struct ivar_t`。最后一段表明 `struct class_ro_t` 中的 `ivars` 指向上面提到的 `ivar_list_t`。

注意：这里保存 `ivars` 的是 `class_ro_t` 而不是 `struct objc_class`。

1. 编译器首先会为 property 自动生成 getter 和 setter ，并放到 `method_list_t` 中。

   ```
   __OBJC_$_INSTANCE_METHODS_THNormalClass:
   	.long	24                              ; 0x18
   	.long	4                               ; 0x4
   	.quad	l_OBJC_METH_VAR_NAME_
   	.quad	l_OBJC_METH_VAR_TYPE_
   	.quad	"-[THNormalClass str1]"
   	.quad	l_OBJC_METH_VAR_NAME_.1
   	.quad	l_OBJC_METH_VAR_TYPE_.2
   	.quad	"-[THNormalClass setStr1:]"
   	.quad	l_OBJC_METH_VAR_NAME_.3
   	.quad	l_OBJC_METH_VAR_TYPE_
   	.quad	"-[THNormalClass str2]"
   	.quad	l_OBJC_METH_VAR_NAME_.4
   	.quad	l_OBJC_METH_VAR_TYPE_.2
   	.quad	"-[THNormalClass setStr2:]"
   ```

2. 由于 property 是成员变量的封装， `ivar_list_t` 中会对其保存对应的成员变量。

   ```
   __OBJC_$_INSTANCE_VARIABLES_THNormalClass:
   	.long	32                              ; 0x20
   	.long	2                               ; 0x2
   	.quad	_OBJC_IVAR_$_THNormalClass._str1
   	.quad	l_OBJC_METH_VAR_NAME_.5
   	.quad	l_OBJC_METH_VAR_TYPE_.6
   	.long	3                               ; 0x3
   	.long	8                               ; 0x8
   	.quad	_OBJC_IVAR_$_THNormalClass._str2
   	.quad	l_OBJC_METH_VAR_NAME_.7
   	.quad	l_OBJC_METH_VAR_TYPE_.6
   	.long	3                               ; 0x3
   	.long	8                               ; 0x8
   ```

3. Objective-C 的每个类有一个 `property_list_t` ，被保存在 `struct class_ro_t` 中。

   ```
   __OBJC_$_PROP_LIST_THNormalClass:
   	.long	16                              ; 0x10
   	.long	2                               ; 0x2
   	.quad	l_OBJC_PROP_NAME_ATTR_
   	.quad	l_OBJC_PROP_NAME_ATTR_.8
   	.quad	l_OBJC_PROP_NAME_ATTR_.9
   	.quad	l_OBJC_PROP_NAME_ATTR_.10
   ```

4. 每个 `property_list_t` 保存的是一个 `struct property_t` 的对象。

   ```
   struct property_t {
       const char *name;
       const char *attributes;
   };
   ```

   ```
   l_OBJC_PROP_NAME_ATTR_:                 ; @OBJC_PROP_NAME_ATTR_
   	.asciz	"str1"
   
   l_OBJC_PROP_NAME_ATTR_.8:               ; @OBJC_PROP_NAME_ATTR_.8
   	.asciz	"T@\"NSString\",C,N,V_str1"
   ```

#### 对成员变量的访问

```objective-c
@interface THNormalClass() {
    int a;
    int b;
}

@end

@implementation THNormalClass

- (int)calAdd {
    return a + b;
}

@end
```

```
FunctionalProgrammingLearn`-[THNormalClass calAdd]:
    0x100b9a4e4 <+0>:  sub    sp, sp, #0x10             ; =0x10 
    0x100b9a4e8 <+4>:  str    x0, [sp, #0x8]
    0x100b9a4ec <+8>:  str    x1, [sp]
    0x100b9a4f0 <+12>: ldr    x8, [sp, #0x8]
->  0x100b9a4f4 <+16>: ldr    w8, [x8, #0x8]
    0x100b9a4f8 <+20>: ldr    x9, [sp, #0x8]
    0x100b9a4fc <+24>: ldr    w9, [x9, #0xc]
    0x100b9a500 <+28>: add    w0, w8, w9
    0x100b9a504 <+32>: add    sp, sp, #0x10             ; =0x10 
    0x100b9a508 <+36>: ret  
```

<img src="https://tva1.sinaimg.cn/large/e6c9d24egy1gzpxu1xwm2j20t707wta3.jpg" alt="image-20220223175619966" style="zoom:90%;" />

`x8` 保存的是 `THNormalClass *` 的地址。`[x8, #0x8]` offset 8 字节找到 a 在内存中的位置。ldr 在这里是内存访问指令。

#### 调用方法

```objective-c
@interface THNormalClass() {
    int a;
    int b;
}

@end

@implementation THNormalClass

- (int)calAdd {
    return [self calMethod];
}

- (int)calMethod {
    a = 1;
    b = 8;
    return a + b;
}

@end
```

```
"-[THNormalClass calAdd]":              ; @"\01-[THNormalClass calAdd]"
	.cfi_startproc
; %bb.0:
	sub	sp, sp, #32                     ; =32
	stp	x29, x30, [sp, #16]             ; 16-byte Folded Spill
	add	x29, sp, #16                    ; =16
	.cfi_def_cfa w29, 16
	.cfi_offset w30, -8
	.cfi_offset w29, -16
	str	x0, [sp, #8]
	str	x1, [sp]
	ldr	x0, [sp, #8]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_@PAGE
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF]
	bl	_objc_msgSend
	ldp	x29, x30, [sp, #16]             ; 16-byte Folded Reload
	add	sp, sp, #32                     ; =32
	ret
```

`adr`：小范围的地址读取指令。将基于 PC  相对偏移的地址值读取到寄存器中。

`adrp`：以 page 为单位大范围的地址读取指令。

`bl`：跳转，还将 bl 的下一条指令的地址保存到寄存器中。

`x1` 保存了 selector 的地址。

#### 调用类方法

```objective-c
@implementation THNormalClass

- (void)calMethod {
    return [THNormalClass calClassMethod];
}

+ (void)calClassMethod {
     
}

@end
```

```
"-[THNormalClass calMethod]":           ; @"\01-[THNormalClass calMethod]"
	.cfi_startproc
; %bb.0:
	sub	sp, sp, #32                     ; =32
	stp	x29, x30, [sp, #16]             ; 16-byte Folded Spill
	add	x29, sp, #16                    ; =16
	.cfi_def_cfa w29, 16
	.cfi_offset w30, -8
	.cfi_offset w29, -16
	str	x0, [sp, #8]
	str	x1, [sp]
	adrp	x8, _OBJC_CLASSLIST_REFERENCES_$_@PAGE
	ldr	x0, [x8, _OBJC_CLASSLIST_REFERENCES_$_@PAGEOFF]
	adrp	x8, _OBJC_SELECTOR_REFERENCES_@PAGE
	ldr	x1, [x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF]
	bl	_objc_msgSend
	ldp	x29, x30, [sp, #16]             ; 16-byte Folded Reload
	add	sp, sp, #32                     ; =32
	ret
```

在上边的汇编代码中，可以看到类方法的调用与成员方法差不多，都是 `_objc_msgSend`。`	adrp	x8, _OBJC_CLASSLIST_REFERENCES_$_@PAGE
	ldr	x0, [x8, _OBJC_CLASSLIST_REFERENCES_$_@PAGEOFF]`，类方法调用多了一步获取全局类指针。

**`adrp x8, 6`**

```
假定当前的 PC 寄存器为 0x10410e324
1.先将 6 的值左移 12 位二进制位则为 0x6000
2.将 PC 寄存器的低12位清零，即 0x10410e324 => 0x10410e000
3.最后将 0x10410e000 和 0x6000 相加给 x8 寄存器。x8 为 0x104114000
```

### block

```objective-c
@implementation THNormalClass

void(^block)(void) = ^(void) {
    NSLog(@"this is a block");
};

- (void)calMethod {
    block();
}

@end
```

![image-20220225163123030](https://tva1.sinaimg.cn/large/e6c9d24egy1gzpuag469uj20s40bamyx.jpg)

<img src="https://tva1.sinaimg.cn/large/e6c9d24egy1gzpxrjlzlxj215m09kjti.jpg" alt="image-20220225163438361" style="zoom:68%;" />

1. 可以看到这是一个 `__NSGlobalBlock__`。

2. ```
   struct Block_layout {
       void * __ptrauth_objc_isa_pointer isa;
       volatile int32_t flags; // contains ref count
       int32_t reserved;
       BlockInvokeFunction invoke;
       struct Block_descriptor_1 *descriptor;
       // imported variables
   };
   ```

   invoke 在内存处于 17-24 字节处。`po 0x00000001028ac090` 得知 invoke ：0x1028aa304。通过 dis -s 来反汇编地址。如下图所示找到了源码中写的 NSLog。

   ```
   (lldb) dis -s 0x00000001028aa304
   FunctionalProgrammingLearn`block_block_invoke:
       0x1028aa304 <+0>:  sub    sp, sp, #0x20             ; =0x20 
       0x1028aa308 <+4>:  stp    x29, x30, [sp, #0x10]
       0x1028aa30c <+8>:  add    x29, sp, #0x10            ; =0x10 
       0x1028aa310 <+12>: str    x0, [sp, #0x8]
       0x1028aa314 <+16>: str    x0, [sp]
       0x1028aa318 <+20>: adrp   x0, 2
       0x1028aa31c <+24>: add    x0, x0, #0xb0             ; =0xb0 
       0x1028aa320 <+28>: bl     0x1028aa668               ; symbol stub for: NSLog
   (lldb) 
   ```

3. `signature ："v8@?0"` 

   [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

### 后记

很简单的在汇编角度对于 OC 一些常用点，比照着 objc4 和 libclosure 的源码探索了下。觉得有必要自己整理个 arm 汇编学习笔记，不求完全记住理解，后边自查也好。