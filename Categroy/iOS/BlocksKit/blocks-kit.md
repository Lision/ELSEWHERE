# BlocksKit 源码解析

头图

## 前言

**Block** 是 Objective-C 可以进行 [Functional programming (FP)](https://en.wikipedia.org/wiki/Functional_programming) 编码的**核心前提**。

毫无疑问，**Block** 使得 Objective-C 的编码**更容易、更快捷、更高效**，尤其在多线程和 [Grand Central Dispatch (GCD)](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html) 的书写时更为明显。

[BlocksKit](https://github.com/BlocksKit/BlocksKit) 希望通过消除一些恼人的问题（并且移除 Block 在某些情况下编码的限制）来推进使用 Block 进行编程。

BlocksKit 原本是由 [Zachary Waldowski](https://github.com/zwaldowski) 和 [Alexsander Akers](https://github.com/a2) 创建，并由前者维护，在 GitHub 早早拿下 6.5k Stars 的骄人成绩。

遗憾的是，在 Issues List 中可以找到 [issue 332](https://github.com/BlocksKit/BlocksKit/issues/332)，其中 Zachary Waldowski 已经明确表示自己没有时间继续维护 BlocksKit 了，这也是 BlocksKit 代码版本止步于 v2.2.5 的原因...

不过瑕不掩瑜，用过 BlocksKit 的同学相信都有 Get 到使用 BlocksKit 的那种令人愉悦的编码体验~

BlocksKit 的内部实现有很多值得我们学习借鉴之处，本文将会针对其中笔者认为有价值的源码实现进行分析解读。

## 索引

- BlocksKit 简介
- BlocksKit - Core
- BlocksKit - UIKit
- BlocksKit - DynamicDelegate
- 总结

## BlocksKit 简介

图片 BlocksKit

BlocksKit 旨在帮助我们去掉 Block 在某些使用场景的局限性和体验不好的地方，提供令人愉悦的使用 Block 的编码体验。

从 BlocksKit 的名字上我们不难看出，其并不只是实现某一功能的**组件**，而是一组主题一致的**套件**。其内部源文件相对较多且杂，其中还包含很多 Categroy ，所以在对 BlocksKit 的划分上我大致遵从其内部的文件结构进行划分，弃掉一些不常用的部分大致划分为三大块：

- Core
- UIKit
- DynamicDelegate

其中 Core 内部是我们在使用 BlocksKit 日常编码时**经常会用到的功能**，UIKit 是一些日常开发工作中高频使用的 **UI 组件的 Categroy 的实现**，而 DynamicDelegate 则负责以**优雅**的方式实现用 Block 替代 CocoaTouch 框架中的常用协议（DataSource，Delegate）代理的实现。

图片 BlocksKit 结构划分

## BlocksKit - Core

BlocksKit - Core 中存放了我们在使用 BlocksKit 日常编码时经常用到的功能实现：

- 对于**集合体的遍历、过滤、映射等高频便捷功能**的支持
- **AssociatedObject** 便捷书写的支持
- 任意 **NSObject** 执行 Block
- **KVO** 便捷书写的支持

### 集合遍历相关便捷功能

跟很多支持 FP 编程的其他语言一样，提供了一些常用的遍历相关的 Block 快捷 API：

- `bk_each:` 简单遍历
- `bk_apply:` 并发遍历
- `bk_match:` 找到第一个符合条件的对象并返回
- `bk_select:` 找到所有符合条件的对象，以数组形式返回
- `bk_reject:` 与 `bk_select:` 相对应，找到所有不符合条件的对象，以数组形式返回
- `bk_map:` 对该数组中的每个对象执行操作后返回一个新的值，以数组形式返回
- `bk_reduce:withBlock:` 初始化一个值，根据 Block 遍历原集合体得到最终值
- `bk_reduceInteger:withBlock:` 同上，适用于 NSInteger
- `bk_reduceFloat:withBlock:` 同上，适用于 CGFloat
- `bk_any:` 是否有条件匹配的对象
- `bk_none:` 是否没有条件匹配的对象
- `bk_all:` 是否全部满足给定条件
- `bk_corresponds:withBlock:` 判断入参数组 list 与该数组是否符合某种联系 block

嘛~ 有没有很眼熟？其实这些集合遍历的便捷方法在 Ruby、Haskell 等语言中也都有对应方法存在。集合遍历相关大致就提供上面列出的 API 了，其内部基本上都是通过 `enumerateObjectsUsingBlock:` 来实现的。

> Note: 想要了解更多关于 `enumerateObjectsUsingBlock:` 与 `for` 以及 `for in` 的区别请[点击这里](https://stackoverflow.com/questions/4486622/when-to-use-enumerateobjectsusingblock-vs-for)。

### AssociatedObject 便捷化

AssociatedObject 作为一项 Objective-C 2.0 Runtime 中加入的新特性，在 OS X Snow Leopard (iOS 4) 之后可用。它的出现允许我们在 Categroy 中加入自定义属性到现有 Class，主要提供了三个方法操作关联对象：

- objc_setAssociatedObject
- objc_getAssociatedObject
- objc_removeAssociatedObjects

BlocksKit 也是使用上述方法实现该功能的，值得一提的是 `objc_AssociationPolicy` 枚举中仅有以下枚举值：

``` objc
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
    OBJC_ASSOCIATION_RETAIN = 01401,
    OBJC_ASSOCIATION_COPY = 01403
};
```

**对于 `weak` 而言，并没有对应的枚举值，BlocksKit 中自己实现了 `weak` 策略的关联。**在 BlocksKit 内部定义了一个 Class：

``` objc
@interface _BKWeakAssociatedObject : NSObject
@property (nonatomic, weak) id value;
@end

@implementation _BKWeakAssociatedObject
@end
```

然后用 `OBJC_ASSOCIATION_RETAIN_NONATOMIC` 关联策略关联 `_BKWeakAssociatedObject` 对象到相应的 `key`，实际操作 `_BKWeakAssociatedObject` 内部 `weak` 引用的 `value` 来实现 `weak` AssociatedObject。

``` objc
+ (void)bk_weaklyAssociateValue:(__autoreleasing id)value withKey:(const void *)key {
	_BKWeakAssociatedObject *assoc = objc_getAssociatedObject(self, key);
	if (!assoc) {
		assoc = [_BKWeakAssociatedObject new];
		[self bk_associateValue:assoc withKey:key];
	}
	assoc.value = value;
}

+ (id)bk_associatedValueForKey:(const void *)key {
	id value = objc_getAssociatedObject(self, key);
	if (value && [value isKindOfClass:[_BKWeakAssociatedObject class]]) {
		return [(_BKWeakAssociatedObject *)value value];
	}
	return value;
}
```

> Note: `__autoreleasing` 表示通过引用（id *）传递的参数，并且在返回时自动释放。

### 任意 NSObject 执行 Block

BlocksKit 的 NSObject+BKBlockExecution 分类中使用 Block 的方式对 `performSelector:` 方法做了修改。它不仅仅在 `perform` 时执行十分迅速，线程安全，使用 GCD 实现异步，并且**每个便捷方法还会返回一个指针，用于在 Block 执行之前取消操作**。

NSObject+BKBlockExecution 内部的便捷方法均依赖 `bk_performBlock:onQueue:afterDelay:` 实现，不论是实例方法还是类方法的实现代码都是一套逻辑：

``` objc
NSParameterAssert(block != nil);
	
__block BOOL cancelled = NO;
	
void (^wrapper)(BOOL) = ^(BOOL cancel) {
	if (cancel) {
		cancelled = YES;
		return;
	}
	if (!cancelled) block();
};
	
dispatch_after(BKTimeDelay(delay), queue, ^{ wrapper(NO); });
	
return [wrapper copy];
```

唯一值得一提的就是 `cancelled` 变量。

``` objc
+ (void)bk_cancelBlock:(id)block {
	NSParameterAssert(block != nil);
	void (^wrapper)(BOOL) = block;
	wrapper(YES);
}
```

在 `bk_cancelBlock:` 函数中调用 `wrapper(YES);` 将 `cancelled` 标记为 `YES` 这样 `wrapper` 在执行时会直接 `return` 实现了对未执行 Block 的可取消。

### KVO 便捷化

如果你想要了解 KVO 的底层是如何实现的，那么恐怕要让你失望了，笔者这里只想简单的介绍一下 BlocksKit 是如何提高我们在开发时编写 KVO 编码效率的。

#### `_BKObserver`

既然是 KVO 便捷化，那么肯定还是基于 KVO 实现的，传统 KVO 实现过程中要实现的那些令人讨厌的方法和固定写法肯定还是绕不开，那么怎么解决这个问题对外仅提供一个简单便捷的 API 呢？

BlocksKit 定义了一个 `_BKObserver` 类作为对 KVO 观察者的具体实现。

``` objc
@interface _BKObserver : NSObject {
	BOOL _isObserving;
}

@property (nonatomic, readonly, unsafe_unretained) id observee;
@property (nonatomic, readonly) NSMutableArray *keyPaths;
@property (nonatomic, readonly) id task;
@property (nonatomic, readonly) BKObserverContext context;

- (id)initWithObservee:(id)observee keyPaths:(NSArray *)keyPaths context:(BKObserverContext)context task:(id)task;

@end
```

> Node: `_BKObserver` 内部 `unsafe_unretained` 引用实际的 `observee`。

这里面的 `BKObserverContext` 实际上是一个枚举：

``` objc
typedef NS_ENUM(int, BKObserverContext) {
	BKObserverContextKey,
	BKObserverContextKeyWithChange,
	BKObserverContextManyKeys,
	BKObserverContextManyKeysWithChange
};
```

`BKObserverContext` 枚举主要起到区分对应 `task` 的 Block 类型的作用，其受 NSKeyValueObservingOptions 的影响。

``` objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
	if (context != BKBlockObservationContext) return;

	@synchronized(self) {
		switch (self.context) {
			case BKObserverContextKey: {
				void (^task)(id) = self.task;
				task(object);
				break;
			}
			case BKObserverContextKeyWithChange: {
				void (^task)(id, NSDictionary *) = self.task;
				task(object, change);
				break;
			}
			case BKObserverContextManyKeys: {
				void (^task)(id, NSString *) = self.task;
				task(object, keyPath);
				break;
			}
			case BKObserverContextManyKeysWithChange: {
				void (^task)(id, NSString *, NSDictionary *) = self.task;
				task(object, keyPath, change);
				break;
			}
		}
	}
}
```

`_BKObserver` 实现了对于 `observee` 的观察同样绕不开 KVO 的系统方法：

``` objc
- (void)startObservingWithOptions:(NSKeyValueObservingOptions)options {
	@synchronized(self) {
		if (_isObserving) return;

		[self.keyPaths bk_each:^(NSString *keyPath) {
			[self.observee addObserver:self forKeyPath:keyPath options:options context:BKBlockObservationContext];
		}];

		_isObserving = YES;
	}
}
```

> Note: `@synchronized(self)` 确保 `_BKObserver` 对象的线程安全，避免了多次 `addObserver` 同一个 `keyPath` 的情况。

对应的还有 `removeObserver:forKeyPath:context:` 方法，这里就不赘述了，不过别忘了在 `_BKObserver` 的 `dealloc` 方法中要对所有的 `keyPath` 注销：

``` objc
- (void)dealloc {
	if (self.keyPaths) {
		[self _stopObservingLocked];
	}
}
```

#### NSObject (BlockObservation)

既然是 KVO 的便捷化实现，那么对于 NSObject 做 Categroy 自然在情理之中，BlockObservation 分类中提供了 KVO 的便捷使用 API，其内部最终都会调用 `bk_addObserverForKeyPaths:identifier:options:context:task:` 方法，只不过是参数上有所差异罢了。

``` objc
- (void)bk_addObserverForKeyPaths:(NSArray *)keyPaths identifier:(NSString *)identifier options:(NSKeyValueObservingOptions)options context:(BKObserverContext)context task:(id)task {
    // 断言参数
	NSParameterAssert(keyPaths.count);
	NSParameterAssert(identifier.length);
	NSParameterAssert(task);

    Class classToSwizzle = self.class;
    // 单例 NSMutableSet，做全局被替换过 dealloc 类的缓存
    NSMutableSet *classes = self.class.bk_observedClassesHash;
    @synchronized (classes) {
        NSString *className = NSStringFromClass(classToSwizzle);
        // 如果单例 NSMutableSet 中没有当前类 Class
        if (![classes containsObject:className]) {
            // 拿到 deallocSelector
            SEL deallocSelector = sel_registerName("dealloc");
            
			__block void (*originalDealloc)(__unsafe_unretained id, SEL) = NULL;
			
            // 这里仅仅引用一下 objSelf，不需要考虑自动置为 nil 的情况
            // 使用 __unsafe_unretained 再合适不过了
			id newDealloc = ^(__unsafe_unretained id objSelf) {
			    // 在 dealloc 中添加删除所有 Block 观察者操作
                [objSelf bk_removeAllBlockObservers];
                
                // 判断此类是否注册过 dealloc 方法
                if (originalDealloc == NULL) { // 没有注册过 dealloc 方法
                    // 用 Block 的入参 objSelf 初始化 objc_super 结构体
                    struct objc_super superInfo = {
                        .receiver = objSelf,
                        .super_class = class_getSuperclass(classToSwizzle)
                    };
                    
                    // 将 objc_msgSendSuper 强转 msgSend
                    void (*msgSend)(struct objc_super *, SEL) = (__typeof__(msgSend))objc_msgSendSuper;
                    // 这样 msgSend 就可以接受 objc_super 作为第一个入参咯
                    // 这样就达到了对当前类父类发送 dealloc 方法的效果
                    msgSend(&superInfo, deallocSelector);
                } else { // 已经注册过 dealloc 方法，执行就是了
                    originalDealloc(objSelf, deallocSelector);
                }
            };
            
            // 创建 newDealloc Block 对应的函数指针 IMP 
            IMP newDeallocIMP = imp_implementationWithBlock(newDealloc);
            
            // 尝试使用 class_addMethod 为当前类添加 dealloc 方法
            // 如果此类已经有 dealloc 方法的实现，class_addMethod 不会覆盖
            if (!class_addMethod(classToSwizzle, deallocSelector, newDeallocIMP, "v@:")) {
                // 添加失败，则获取当前类的 dealloc 方法
                Method deallocMethod = class_getInstanceMethod(classToSwizzle, deallocSelector);
                
                // 拿到当前方法 dealloc 的 IMP
                // 在设置新的实现之前，我们需要存储原始实现，以防在设置时调用方法
                originalDealloc = (void(*)(__unsafe_unretained id, SEL))method_getImplementation(deallocMethod);
                
                // 用新的 dealloc IMP 绑定 dealloc 方法
                // 我们需要再次存储原始实现，以防它刚刚更改
                originalDealloc = (void(*)(__unsafe_unretained id, SEL))method_setImplementation(deallocMethod, newDeallocIMP);
            }
            
            // 完成方法替换后，将当前类加入全局替换类缓存
            [classes addObject:className];
        }
    }

    // 通过 self 和入参构建 _BKObserver
    // 通过 _BKObserver 实现具体的观察
	NSMutableDictionary *dict;
	_BKObserver *observer = [[_BKObserver alloc] initWithObservee:self keyPaths:keyPaths context:context task:task];
	[observer startObservingWithOptions:options];

	@synchronized (self) {
	    // 这个 dict 也是通过 runtime 关联的
		dict = [self bk_observerBlocks];
		
        // lazyload
		if (dict == nil) {
			dict = [NSMutableDictionary dictionary];
			[self bk_setObserverBlocks:dict];
		}
	}
    
    // 将 _BKObserver 对象实例 retain 到 dict 当中
	dict[identifier] = observer;
}
```

Emmmmm...代码比较长，所以加了详细的注释，不过没有什么特别难懂的点，简单来说，就是 Hook 了 `dealloc` 方法，加入了注销 KVO 观察者的操作而已~

至于存储 `_BKObserver` 的字典其实是通过 AssociatedObject 关联的：

``` objc
- (void)bk_setObserverBlocks:(NSMutableDictionary *)dict {
	[self bk_associateValue:dict withKey:BKObserverBlocksKey];
}

- (NSMutableDictionary *)bk_observerBlocks {
	return [self bk_associatedValueForKey:BKObserverBlocksKey];
}
```

