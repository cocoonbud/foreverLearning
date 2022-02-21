#### 前言

很早前，看到过一个关于基于 iOS 响应者链条和事件处理机制的总结性文章，评论里有说可以通过这个机制进行简单的 UI 间事件处理。当时没太在意，未深入了解。后来自己总结此块内容时，写了这么个小工具。

如 subview -> vcview -> vc。通常我们是把 subview 中的事件传递到 vc，在 vc 中处理。基本的通过通知、代理、block 等进行传递。但是往往会遇到多 view 层级，往往我们得一层一层的向上传递。

有没有什么简单的办法可以让我们省去写这些传递代码？

#### 基础知识回顾

关于响应者链条和事件处理机制在此不展开细说。简单回顾两条。

* 响应者链的链路是: 
  由离用户最近的 view 向系统传递。
  ``` initial view –> super view –> ….. –> view controller –> window –> Application –> AppDelegate```

* 传递链条：

  由系统向离用户最近的 view 传递。

  ```前台 App –> window –> root view –> …… –> view```

#### 原理

1. 在响应者链条上，不断循环判断 responder 有没有实现接收事件处理的协议。
2. 在 View 的视图层级内，按照传递链条，不断递归判断是否有实现接受事件处理的协议。

配合三个枚举值（忽略事件并继续传递、处理事件并继续传递、处理事件并结束传递）

#### 实现

##### 定义协议

```objective-c
@protocol THUIEventBusProtocol <NSObject>

@optional

/// receiving upward-passed events with event name
/// @param eventName event name
/// @param data data
/// @param callback after handle callback
- (THUIEventResult)receivingUpwardPassedEventsWithEventName:(nonnull NSString *)eventName data:(nullable id)data callback:(void(^_Nullable)(id _Nullable otherData))callback;


/// receiving downward-passed events with event name
/// @param eventName event name
/// @param data data
/// @param callback after handle callback
- (THUIEventResult)receivingDownwardPassedEventsWithEventName:(nonnull NSString *)eventName data:(nullable id)data callback:(void(^_Nullable)(id _Nullable otherData))callback;

@end
```

##### 两个核心处理方法

1. 按照响应者链路向上传递事件 （如 view 传递给其所在的 vc）

   ```objective-c
   + (void)passingEventsUpwardsWithEventName:(nonnull NSString *)eventName
                                   responder:(nonnull UIResponder *)responder
                                        data:(nullable id)data
                                    callback:(void(^)(id _Nullable otherData))callback {
       if (eventName.length < 1 || !responder || ![responder isKindOfClass:[UIResponder class]]) return;
    
       UIResponder <THUIEventBusProtocol> *curResponder = (UIResponder <THUIEventBusProtocol> *)responder.thNextResponder;
       
       while (curResponder && ![curResponder isKindOfClass:[UIWindow class]] && ![curResponder isKindOfClass:[UIApplication class]]) {
           
           NSLog(@"🌺 cur responder is %@", NSStringFromClass([curResponder class]));
           
           if ([curResponder conformsToProtocol:@protocol(THUIEventBusProtocol)]) {
               if ([curResponder respondsToSelector:@selector(receivingUpwardPassedEventsWithEventName:data:callback:)]) {
                   
                   THUIEventResult res = [curResponder receivingUpwardPassedEventsWithEventName:eventName
                                                                                           data:data
                                                                                       callback:callback];
                   
                   if (res == THUIEventResultHandleAndStop) break;
               }
           }
           curResponder = (UIResponder <THUIEventBusProtocol> *)curResponder.thNextResponder;
       }
   }
   ```

2. 按传递链条向下传递 （如 vc 传递给其所有的 subview）

   ```objective-c
   + (void)passingEventsDownWithEventName:(nonnull NSString *)eventName
                                responder:(nonnull UIResponder *)responder
                                     data:(nullable id)data
                                 callback:(void(^)(id _Nullable otherData))callback {
       if (eventName.length < 1 || !responder || ![responder isKindOfClass:[UIResponder class]]) return;
       
       NSLog(@"🍊 cur responder is %@", NSStringFromClass([responder class]));
       
       THUIEventResult res = THUIEventResultIgnoreAndContinue;
               
       if ([responder isKindOfClass:[UIViewController class]]) {
   				// vc
           UIViewController <THUIEventBusProtocol> *responderVC = (UIViewController <THUIEventBusProtocol> *)responder;
           
           if ([responderVC conformsToProtocol:@protocol(THUIEventBusProtocol)]
               && [responderVC respondsToSelector:@selector(receivingDownwardPassedEventsWithEventName:data:callback:)]) {
               
               res = [responderVC receivingDownwardPassedEventsWithEventName:eventName data:data callback:callback];
               
               if (res == THUIEventResultHandleAndStop) return;
           }
           
           for (UIViewController *childVC in responderVC.childViewControllers) {
              // 子 vc
              [self passingEventsDownWithEventName:eventName responder:childVC data:data callback:callback];
           }
           
         	// 子 view
           [THUIEventBus handleViewWithEventName:eventName responder:responderVC.view data:data callback:callback];
       }
       
       // view
       if ([responder isKindOfClass:[UIView class]]) {
   				...
       }
   }
   ```

   通过 eventName 来区分接收到的事件如何处理，忽略并继续传递？处理并继续传递？处理并停止传递？

   #### 其他

   ##### 给 UIResponder 加类扩展，加多一个 thNextResponder。

   可主动设置 nextResponder，控制事件的传递。

   ##### 方法映射表

   如以下场景：

   在 vc 中

   ```oc
   - (THUIEventResult)receivingUpwardPassedEventsWithEventName:(nonnull NSString *)eventName data:(nullable id)data callback:(void(^)(id _Nullable otherData))callback {
       return THUIEventResultHandleAndStop;
   }
   ```

   我们在这个方法中接受子 view 传递过来的事件，依据 eventName 可能有不同的处理逻辑。此时首先会想到 if else。若事件较多，可考虑写一个方法映射表。

   ```objective-c
   - (NSDictionary <NSString *, NSInvocation *> *)methodDict {
       if (_methodDict == nil) {
           _methodDict = @{
               kUIEventA : [self createInvocationWithSelector:@selector(handleEventA:)],
               kUIEventB : [self createInvocationWithSelector:@selector(handleEventB:)],
               kUIEventC : [self createInvocationWithSelector:@selector(handleEventC:)]
           };
       }
       return _methodDict;
   }
   ```

   ```objective-c
   - (NSInvocation *)createInvocationWithSelector:(SEL)selector {
       NSMethodSignature *signature = [[self class] instanceMethodSignatureForSelector:selector];
       
       if (signature == nil) {
           NSString *info = [NSString stringWithFormat:@"-[%@ %@] : unrecognized selector sent to instance", [self class],
                             NSStringFromSelector(selector)];
           @throw [[NSException alloc] initWithName:@"No such method" reason:info userInfo:nil];
           return nil;
       }
       
       NSInvocation *invo = [NSInvocation invocationWithMethodSignature:signature];
       
       invo.target = self;
       invo.selector = selector;
       return invo;
   }
   ```

#### [项目地址](https://github.com/cocoonbud/THUIEventBus)

