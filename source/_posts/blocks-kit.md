---
title: ⌈OC⌋简而繁的BlocksKit
---

---
意如分类标题(这个博客主题貌似没有，😂)那样，笔者打算开始探究框架与源码。毫无疑问，这其中会遇到各种各样的挑战，但是我觉得我们应尽早走出这一步，不然就错过了很多的精彩。也许这精彩是更加开阔的程序视野，亦或是逻辑思维与编程能力的提升，这其中对耐心与意志的磨炼绝对会让人十分“酸爽”。

而这简而繁`BlocksKit`就成为了第一道`菜`，为什么呢？因为之前对它的认知就是对系统`API`的`block`方式调用的高度封装，这是它简单的使用特性。然而它的灵魂——动态代理让我深切感受到了框架的设计哲学：`把简洁留给别人，把复杂留给自己`。不信？那就接着往下看！
<!-- more -->

## 简单的API使用与探究
你或许诟病过`Target-Action`响应模式代码的编写，又或者对代理模式爱的深沉。`BlocksKit`给你带来了福音，给按钮添加监听方法是这么写的：(原谅我这里把`Objective-C`代码标记为`C`，因为这个博客主题竟然识别不了`Objective-C`，💔)

```C
[button bk_addEventHandler:^(id sender) {
        NSLog(@"点了我");
} forControlEvents:UIControlEventTouchUpInside];
```
这是`UIControl`添加的分类`BlocksKit`中的一个方法，以下是它的实现：

```C
- (void)bk_addEventHandler:(void (^)(id sender))handler forControlEvents:(UIControlEvents)controlEvents {
	//首先判断是否有事件的响应
	NSParameterAssert(handler);
	
	//动态获取对象绑定到的事件，对于只有一个事件的实例，此处的event自然是nil
	NSMutableDictionary *events = objc_getAssociatedObject(self, BKControlHandlersKey);
	if (!events) {
		events = [NSMutableDictionary dictionary];
		//如果当前当前为首次给对象添加事件，则新建一个事件字典，并动态关联到对象
		objc_setAssociatedObject(self, BKControlHandlersKey, events, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
	}

	//使用事件在枚举中的值作为键，取出该事件的处理者
	NSNumber *key = @(controlEvents);
	NSMutableSet *handlers = events[key];
	if (!handlers) {
		handlers = [NSMutableSet set];
		events[key] = handlers;
	}
	
	//真正的目标对象，它有两个属性：handler和controlEvent，初始化方法就是对这两者赋值
	BKControlWrapper *target = [[BKControlWrapper alloc] initWithHandler:handler forControlEvents:controlEvents];
	//添加对应时间的响应者
	[handlers addObject:target];

	//target的invoke:方法就是调用自身的handler，并将本身作为参数传入
	[self addTarget:target action:@selector(invoke:) forControlEvents:controlEvents];
}

```
这里之所以采用字典的形式来存储事件与响应者，是为了后面能够一次性移除某个事件的响应者和判断某个事件是否有响应者。具体可以看看`bk_removeEventHandlersForControlEvents:`和`bk_hasEventHandlersForControlEvents:`这两个方法，这里就不再赘述。

代理模式的实现，用到了本框架的核心模块：<font color=#893212>动态代理</font>。后面会用相当篇幅解读，这里先看看其他有趣的东西，比方说延时手势。关于构造实际响应目标和按钮大同小异，这里主要看看如何让手势延时响应。手势响应时都会触发一个位于`UIGestureRecognizer`分类中的`bk_handleAction:`方法，以下仅贴出关于延时的实现：

```C
- (void)bk_handleAction:(UIGestureRecognizer *)recognizer {
	//部分省略
	dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC));
	
    //在指定时间后，将任务加入到主队列中等待执行，而非是任务在主队列中等待指定时间长度后执行
	dispatch_after(popTime, dispatch_get_main_queue(), block);
}
```
## `特色菜`味道就是好
<font color=330C0580 size=4>weak对象关联</font>

如果你使用过分类（`Category`）的话，你肯定知道它不能为对象直接新增属性。因为对象的本质是结构体，在编译时它的内存空间大小就已经确定了。它有一个成员变量是`methodLists`，它是这样的一个类型：`struct objc_method_list **`，即指向`objc_method_list`类型的结构体二级指针。如果是系统固有的方法列表，那么它的大小就是固定的。当我们使用分类增加方法时，本质上就是通过修改`*methodLists`值来指定新的内存地址以容下新增的方法（方法本身也是一个结构体）。

而`Runtime`的对象关联（`AssociateObject`）解决了使用分类不能新增属性的问题，其实就涉及到了两个函数，因此极易上手：

```C
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)

id objc_getAssociatedObject(id object, const void *key)
```
系统提供的关联策略有如下几种：

```C
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
	// 属性名key直接指向所关联的对象，不增加其引用计数，相当于__unsafe_unretained，相当于修饰属性的assign, atomic
    OBJC_ASSOCIATION_ASSIGN = 0, 
    
    // 相当于用nonatomic、retain修饰属性      
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, 

	// 相当于copy、nonatomic
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3, 

	// 相当于retain, atomic
    OBJC_ASSOCIATION_RETAIN = 01401,   

	// 相当于copy, atomic
    OBJC_ASSOCIATION_COPY = 01403    
};

```
不难看出，没有`weak`这一关联策略。那么我们来看看`BlocksKit`是如何实现的：

```C
- (void)bk_weaklyAssociateValue:(__autoreleasing id)value withKey:(const void *)key {
	_BKWeakAssociatedObject *assoc = objc_getAssociatedObject(self, key);
	if (!assoc) {
		assoc = [_BKWeakAssociatedObject new];
		[self bk_associateValue:assoc withKey:key];
	}
	assoc.value = value;
}
```

```C
- (id)bk_associatedValueForKey:(const void *)key
{
	id value = objc_getAssociatedObject(self, key);
	if (value && [value isKindOfClass:[_BKWeakAssociatedObject class]]) {
		return [(_BKWeakAssociatedObject *)value value];
	}
	return value;
}

```

```C
@interface _BKWeakAssociatedObject : NSObject

@property (nonatomic, weak) id value;

@end
```
不难看出，作者使用了一个中间对象，它持有一个`weak`类型的属性`value`来存储实际所赋的值。在获取关联对象时就判断其是否为`_BKWeakAssociatedObject`类型的对象，如果是就返回该对象`value`的属性值。

<font color=330C0580 size=4>取消block执行</font>

有时我们需要在指定时间后追加一个`block`到队列中，而且它只在条件满足时才会执行，也就是说这个`block`的执行可以在未开始前取消。如果`GCD`支持的话，用`dispatch_block_cancel(dispatch_block_t block)`这个函数取消即可。但是它的使用声明是`__OSX_AVAILABLE_STARTING(__MAC_10_10, __IPHONE_8_0)`，而`BlocksKit`出现是几年前的事了，所以作者完全应当考虑到`GCD`不支持的情况，所以就有了这段条件编译的指令：

```C
#if (defined(__IPHONE_OS_VERSION_MAX_ALLOWED) && __IPHONE_OS_VERSION_MAX_ALLOWED >= 80000) || (defined(__MAC_OS_X_VERSION_MAX_ALLOWED) && __MAC_OS_X_VERSION_MAX_ALLOWED >= 1010)
// 两种情况下的宏值一样是一个奇怪的问题，在github这个框架的issue上也有人提出了这个疑问，但没人回复。
#define DISPATCH_CANCELLATION_SUPPORTED 1
#else
#define DISPATCH_CANCELLATION_SUPPORTED 1
#endif
```
不过这都是细枝末节，权当是作者手误，在这里姑且认为这个宏是有正确意义的。那么就来看看怎么使用一个可以取消的`block`：

```C
NSLog(@"start");

BKCancellationToken token = [self bk_performAfterDelay:10.0f usingBlock:^(id  _Nonnull obj) {
	NSLog(@"haha");
}];
    
//[UIViewController bk_cancelBlock:token];

//未取消注释的打印信息：
//2017-08-16 11:06:44.058 BlocksKit的block中途取消测试[1536:50956] start

//取消注释的打印信息：
//2017-08-16 11:07:54.038 BlocksKit的block中途取消测试[1565:51979] start
//2017-08-16 11:08:04.040 BlocksKit的block中途取消测试[1565:51979] haha

```
可以看到`block`确实是可以被取消执行的。我们注意到这里有一个`token`，嗯？进入方法内部一探究竟。第一个方法有这样的调用层级关系：

`bk_performAfterDelay:usingBlock:
 |__bk_performOnQueue:afterDelay:usingBlock:
 ...|__BKDispatchCancellableBlock()`

最后这个`BKCancellationToken`类型的`token`，是这样返回的：

```C
static id <NSObject, NSCopying> BKDispatchCancellableBlock(dispatch_queue_t queue, NSTimeInterval delay, void(^block)(void)) {
    dispatch_time_t time = BKTimeDelay(delay);
    
#if DISPATCH_CANCELLATION_SUPPORTED
    if (BKSupportsDispatchCancellation()) {
        dispatch_block_t ret = dispatch_block_create(0, block);
        // 在指定时间后将任务追加到队列中
        dispatch_after(time, queue, ret);
        return ret;
    }
// 另一个分支后面贴出
```
这里就可以知道`BKCancellationToken`实际上就是`id <NSObject, NSCopying>`，在`NSObject+BKBlockExecution.h`文件中可以印证这一说法，它本质上就是接下来需要执行的`block`。作者使用了双重保险来验证`GCD`是否支持取消`block`执行，函数`BKSupportsDispatchCancellation()`的实现是这样的：

```C
// NS_INLINE表示函数为内联函数，在编译时，函数的实现代码就会被拷贝到调用处，然后编译在同一个可执行文件中。这样可以提高函数的调用效率，但仅适用于体积小且逻辑简单的函数，否则可执行文件就可能变得很大。
NS_INLINE BOOL BKSupportsDispatchCancellation(void) {
#if DISPATCH_CANCELLATION_SUPPORTED
    return (&dispatch_block_cancel != NULL);
#else
    return NO;
#endif
}
```
第二次验证实际上就是寻找函数`dispatch_block_cancel`的入口地址，如果是空值`NULL`（相当于`OC`中的`nil`），则表示不支持。

接着看看`BlocksKit`如何自实现可取消执行的`block`：

```C
#endif  
    __block BOOL cancelled = NO;
    void (^wrapper)(BOOL) = ^(BOOL cancel) {
        if (cancel) {
            cancelled = YES;
            return;
        }
        if (!cancelled) block();
    };
    
    dispatch_after(time, queue, ^{
        wrapper(NO);
    });
    
    return wrapper;
}
```
作者巧妙的用了一个`__block`的`cacelled`标记，标记无效时才执行`block`。默认情况下就传入`NO`，使其能正常工作。将要取消时传入`YES`即可，以下就是`bk_cancelBlock`的实现：

```C
+ (void)bk_cancelBlock:(id <NSObject, NSCopying>)block {
    NSParameterAssert(block != nil);
    
#if DISPATCH_CANCELLATION_SUPPORTED
    if (BKSupportsDispatchCancellation()) {
        dispatch_block_cancel((dispatch_block_t)block);
        return;
    }
#endif
    
    void (^wrapper)(BOOL) = (void(^)(BOOL))block;
    wrapper(YES);
}

```
## 食髓知味，神奇的动态代理
看到这里相信你已经了解了`BlocksKit`的一些基本使用的原理了，也肯定它们难不倒聪明的你。那么接下来的两个模块`KVO`和`动态代理`请你保持足够耐心（尤其是后者），期望你在阅读完之后依然能够轻松地呼出一口气，证明你确实理解了和笔者我真正的讲解到位了。

嗯，立个`FLAG`：***前方高能预警***！

<font color=330C0580 size=4>包装KVO</font>

导入了这个框架，你就可以这么使用`KVO`，十分简洁：

```C
[self.view bk_addObserverForKeyPath:@"backgroundColor" options:NSKeyValueObservingOptionNew task:^(id obj, NSDictionary *change) {
	NSLog(@"target:%@ change:%@", obj, change);
}];
```
以下是`NSObject+BKBlockObservation.h`中提供的所有关于`KVO`的接口：

```C
@interface NSObject (BlockObservation)

// 不指定标识符添加观察者，方法会返回一个标识符用作取消观察者的参数，默认是进程的标识信息。
- (NSString *)bk_addObserverForKeyPath:(NSString *)keyPath task:(void (^)(id target))task;
- (NSString *)bk_addObserverForKeyPaths:(NSArray *)keyPaths task:(void (^)(id obj, NSString *keyPath))task;
- (NSString *)bk_addObserverForKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options task:(void (^)(id obj, NSDictionary *change))task;
- (NSString *)bk_addObserverForKeyPaths:(NSArray *)keyPaths options:(NSKeyValueObservingOptions)options task:(void (^)(id obj, NSString *keyPath, NSDictionary *change))task;

- (void)bk_addObserverForKeyPath:(NSString *)keyPath identifier:(NSString *)token options:(NSKeyValueObservingOptions)options task:(void (^)(id obj, NSDictionary *change))task;

// 添加指定键路径数组、标识和监听选项的观察者
- (void)bk_addObserverForKeyPaths:(NSArray *)keyPaths identifier:(NSString *)token options:(NSKeyValueObservingOptions)options task:(void (^)(id obj, NSString *keyPath, NSDictionary *change))task;

// 移除指定标识和键路径的观察者
- (void)bk_removeObserverForKeyPath:(NSString *)keyPath identifier:(NSString *)token;

- (void)bk_removeObserversWithIdentifier:(NSString *)token;

// 移除所有的观察者
- (void)bk_removeAllBlockObservers;

@end
```
<font color=#478990>大多框架都采用提供多个参数个数不一接口，而实质上接口的实现都是调用自身的一个参数最为完备的方法来克服`objc`不支持默认参数的问题和满足类自身的封装性。</font>在这里添加监听者的方法最终就进入了这样的一段代码：

```C
- (void)bk_addObserverForKeyPaths:(NSArray *)keyPaths identifier:(NSString *)identifier options:(NSKeyValueObservingOptions)options context:(BKObserverContext)context task:(id)task {
	// 保证参数的完整性
	NSParameterAssert(keyPaths.count);
	NSParameterAssert(identifier.length);
    // task就是block
	NSParameterAssert(task);
	// 这个方法实现较长，为了能够更加细致地分析，后面会慢慢贴出逻辑结构
```

```C
	Class classToSwizzle = self.class;
	
	// 使用一个单例的集合来存放已经Swizzling过的类，具体就是拌和dealloc的实现。对于一个类来说，dealloc方法拌和一次就可以了。这里不关心集合中元素的顺序，故用NSMutableSet就可以了。
	NSMutableSet *classes = self.class.bk_observedClassesHash;

	// 使用互斥锁保证线程安全，防止同时对classes操作造成不可预期的结果
	@synchronized (classes) {
		NSString *className = NSStringFromClass(classToSwizzle);
		if (![classes containsObject:className]) {
```
下面进入`Method Swizzling`关键部分：

```C
			SEL deallocSelector = sel_registerName("dealloc");
            
			__block void (*originalDealloc)(__unsafe_unretained id, SEL) = NULL;
            
			id newDealloc = ^(__unsafe_unretained id objSelf) {
				// 拌和dealloc方法的核心目的就是希望在dealloc之前移除所有的观察者
		        [objSelf bk_removeAllBlockObservers];
                
                // 如果当前类中没有实现dealloc方法，就动态发送消息给父类。对于那些自定义的类，方法能够添加成功，originalDealloc就一直是NULL
	            if (originalDealloc == NULL) {
		            struct objc_super superInfo = {
	                .receiver = objSelf,
	                .super_class = class_getSuperclass(classToSwizzle)
	            };
                    
		            void (*msgSend)(struct objc_super *, SEL) = (__typeof__(msgSend))objc_msgSendSuper;
            
		            msgSend(&superInfo, deallocSelector);
	            } else {
		            // 如果有，直接调用
		            originalDealloc(objSelf, deallocSelector);
	            }
	        };
            
            // 感觉这里的命名有点晕，上面定义的newDealloc实际上是用来构造IMP的block，这里就是定义新的dealloc实现
	        IMP newDeallocIMP = imp_implementationWithBlock(newDealloc);
            
            // 对于系统类来说，肯定不能添加成功。因为NSObject已经有dealloc方法，默认实现就是调用父类。ARC中不允许出现[super dealloc]，因为运行时会自动完成。
           // 自定义类可以添加成功，因为自定义类中可以覆盖父类方法实现。从这个角度来看，这个操作是可以成功的。
           // 由这个方法官方解释可知给类添加的方法必须接收两个参数，也就是self和_cmd两个隐式参数，故需要指定类型为v@:
           //v@: 表示方法类型为无返回值接收两个参数：一个对象和一个selector。
			if (!class_addMethod(classToSwizzle, deallocSelector, newDeallocIMP, "v@:")) {
                // 添加不成功，说明之前已经有方法的实现了，就把之前的方法取出来。
				Method deallocMethod = class_getInstanceMethod(classToSwizzle, deallocSelector);
                
                // We need to store original implementation before setting new implementation
                // in case method is called at the time of setting.
                // 将之前的实现存储起来，防止在设置过程中方法被调用。
				originalDealloc = (void(*)(__unsafe_unretained id, SEL))method_getImplementation(deallocMethod);
                
                // We need to store original implementation again, in case it just changed.
                // 设置新的方法实现时，会返回之前的实现。这里再次赋值是为了防止它已经被其他调用者改变了（比如用拌和添加了其他逻辑），也就是保证originalDealloc是设置之前的最新的实现
				originalDealloc = (void(*)(__unsafe_unretained id, SEL))method_setImplementation(deallocMethod, newDeallocIMP);
			}
            
            //添加成功后，就把这个类标记为已经拌合了dealloc方法
			[classes addObject:className];   
		}
	}
```
实例化实际的观察者：

```C
	_BKObserver *observer = [[_BKObserver alloc] initWithObservee:self keyPaths:keyPaths context:context task:task];

	//实现如下：
	- (id)initWithObservee:(id)observee keyPaths:(NSArray *)keyPaths context:(BKObserverContext)context task:(id)task {
	 if ((self = [super init])) {
		 _observee = observee;
		 _keyPaths = [keyPaths mutableCopy];
		 _context = context;
		 _task = [task copy];
	 }
		 return self;
	}
```

```
	[observer startObservingWithOptions:options];

	// 实现如下：
	- (void)startObservingWithOptions:(NSKeyValueObservingOptions)options {
		@synchronized(self) {
			// 防止同一个对象添加多个相同的观察者
			if (_isObserving) return;

			[self.keyPaths bk_each:^(NSString *keyPath) {
				// 调用系统的KVO
				[self.observee addObserver:self forKeyPath:keyPath options:options context:BKBlockObservationContext];
			}];

			_isObserving = YES;
		}
	}

```

```C
	NSMutableDictionary *dict;
		@synchronized (self) {
			dict = [self bk_observerBlocks];

			if (dict == nil) {
				dict = [NSMutableDictionary dictionary];
				[self bk_setObserverBlocks:dict];
			}
		}
	// 以标识符为键，存储观察者信息。这样移除指定标识符的观察者，把该键对应的值移除就可以了。
	dict[identifier] = observer;
}
```
在观察的键值改变后，就根据监听选项来回调`block`：

```C
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {

	// 保证只回调使用`startObservingWithOptions`方法的添加观察者，如此才能有正确的监听选项匹配，task的参数能够对应起来。
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

// 移除观察者的代码相对简单，限于篇幅，这里就不过多唠叨了。

```
<font color=330C0580 size=4>动态代理</font>

作为`BlocksKit`的灵魂所在，逻辑结构和代码的嵌套层次并不是三下五除二就能明白和理清的。既然你已经看到了这里，那么请相信你是一个很有耐心的人，为你鼓掌！那么我们开始本文的最后一个模块，也是干货最多的一个部分！

我们先思考静态的代理是如何实现的，无论是`delegate`还是`dataSource`！其实总共分三步，注意不是`将大象装冰箱的那三步`，而是：

 1. 指定代理对象；
 2. 代理对象遵守协议；
 3. 代理对象实现协议。

当然第一步和第二步的顺序不是那么界限分明，不过这无关紧要，重要的是系统会在对象需要执行代理方法时做出怎样的响应。其实过程很简单：

 1. 查看该对象的代理对象，如果没有，则忽略这一系列的消息。如果有，就进入下一步；
 2. 查看代理对象是否遵守协议，对于有必选方法的协议，如果代理对象没有遵守协议，则程序运行直接崩溃；（如`UITableView`）
 3. 查看代理对象是否实现协议中的必选方法，没有程序也会在运行时崩溃。（如`UITableView`）

在文章开篇的时候，我们就已经知道了在`BlocksKit`的强力驱动下，我们只需要做什么就可以实现代理模式，这里以及后面的分析都以`UIWebView`为例：

```C
//调用属性的setter方法
[webView bk_setDidStartLoadBlock:^(UIWebView *web) {
	//..        
}];
    
[webView bk_setDidFinishLoadBlock:^(UIWebView *web) {
	//..        
}];
    
[webView bk_setDidFinishWithErrorBlock:^(UIWebView *web, NSError *error) {
	NSLog(@"error: %@", error.localizedDescription);
}];
```
几乎可以说是零成本！然而这只是表面的简单而已，它的背后实则隐藏着各种复杂各种蒙圈各种......而文章到这里的目的就是希望将它们抽丝剥茧，化整为零，化繁为简，那么就来看看作者是怎样在背后完成动态代理的吧！（注：以下内容用`UIWebView`作为分析实例，余者（系统对象）除了代理方法不同之外，实现过程是完全一样的。）

<font color=330C0580 size=3>- - 找对象，找动态代理对象，上BlocksKit - -</font>

如果你注意看了框架的源文件，就会发现`UIWebView`、`UITextField`和`UIActionSheet`等原生视图对象都有`BlocksKit`的分类，并且这里面还添加的是一系列属性。这是你可能就会想到`属性的本质 = getter + setter + iVar`，然后进入实现文件中看看有木有熟悉的动态关联代码。于是你就发现了作者的目的是<font color=#478990>动态生成属性的存取方法</font>，然后就是喜闻乐见的`+ load`：

```C
+ (void)load {
    // 以下是公用的注册动态代理和链接代理方法，自动释放池会让一个文件中调用它们产生的中间对象提前释放，防止对其他文件造成影响。
	@autoreleasepool {
		[self bk_registerDynamicDelegate];
		// 链接代理方法的调用省去，节约篇幅
	}
}
```
众所周知，这里是进行`Method Swizzling`最适合的地方。故作者自然`顺遂`了我们的心意，在这里面完成`delegate`的`getter`和`setter`的拌合：

```C
+ (void)bk_registerDynamicDelegate {
    // 根据某一协议注册delegate
    // 第二个参数就是获取当前对象的代理协议对象，代码不多，可自行查看
	[self bk_registerDynamicDelegateNamed:@"delegate" forProtocol:a2_delegateProtocol(self)];
}
```

```C
+ (void)bk_registerDynamicDelegateNamed:(NSString *)delegateName forProtocol:(Protocol *)protocol {
    //self = UIWebView类对象 protocol = UIWebViewDelegate delegateName = @"delegate"
    
    // 取得当前协议的代理信息映射表（可将其看作字典），传入YES表示没有则创建，并绑定在self上，用来记录setter和getter的拌合状态，避免多次拌合。
	NSMapTable *propertyMap = [self bk_delegateInfoByProtocol:YES];
	
    //结构体：存储getter、setter、a2_setter
	A2BlockDelegateInfo *infoAsPtr = (__bridge void *)[propertyMap objectForKey:protocol];
	
	//正常情况下，一个类只会调用一次注册动态代理的方法。
	if (infoAsPtr != NULL) { return; }

	const char *name = delegateName.UTF8String;
	objc_property_t property = class_getProperty(self, name);
    //setter: setDelegate:
	SEL setter = setterForProperty(property, name);
    
    //a2_setter: a2_SetDelegate:
	SEL a2_setter = prefixedSelector(setter);
    
    //getter: delegate
	SEL getter = getterForProperty(property, name);

    //初始化block代理信息
	A2BlockDelegateInfo info = {
		setter, a2_setter, getter
	};

    //将`delegate`的setter、getter、a2_Setter保存在对应的协议名下面，用mapTable存储
	[propertyMap setObject:(__bridge id)&info forKey:protocol];
    
    //将协议下的代理信息取出来
	infoAsPtr = (__bridge void *)[propertyMap objectForKey:protocol];

    // 以下的核心目的是拌合对象本身delegate属性的getter、setter
    
    //构建新的setter的实现，block接收一个NSObject被代理对象和id类型的代理对象
    // 当指定代理对象时，才会调用这个block，也就是你自己设置对象的delegate。
	IMP setterImplementation = imp_implementationWithBlock(^(NSObject *delegatingObject, id delegate) {
        // realDelegate是让用户可以设置代理对象
		A2DynamicDelegate *dynamicDelegate = getDynamicDelegate(delegatingObject, protocol, infoAsPtr, YES);
		if ([delegate isEqual:dynamicDelegate]) {
            // 如果设置realDelegate就是动态代理对象本身，就将realDelegate设置为nil，即依旧使用动态代理自身响应消息
			delegate = nil;
		}
		dynamicDelegate.realDelegate = delegate;
	});

	// **重要**：将原有的setDelegate:实现替换成上面构建的新实现，而a2_setter即a2_SetDelegate：实现就变成了系统实现：xxx.delegate = xxx。正常情况下，能够交换成功。
	if (!swizzleWithIMP(self, setter, a2_setter, setterImplementation, "v@:@", YES)) {
        //bzero，置字节字符串s的前n个字节为零，即清除该协议下的代理信息
		bzero(infoAsPtr, sizeof(A2BlockDelegateInfo));
		return;
	}

    //对于UIWebView来说，是能够响应getter：delegate。以下情况是为了适配自定义代理模式。
	if (![self instancesRespondToSelector:getter]) {
		IMP getterImplementation = imp_implementationWithBlock(^(NSObject *delegatingObject) {
			return [delegatingObject bk_dynamicDelegateForProtocol:a2_protocolForDelegatingObject(delegatingObject, protocol)];
		});

		addMethodWithIMP(self, getter, NULL, getterImplementation, "@@:", NO);
	}
}
```
到此，系统依然不知道代理对象是谁，这里的一系列操作是后续流程的大前提。

<font color=330C0580 size=3>- - 不只牵线搭桥，还包装配到家 - -</font>

贴上`+load`中省略的那部分：

```C
[self bk_linkDelegateMethods:@{
			@"bk_shouldStartLoadBlock": @"webView:shouldStartLoadWithRequest:navigationType:",
			@"bk_didStartLoadBlock": @"webViewDidStartLoad:",
			@"bk_didFinishLoadBlock": @"webViewDidFinishLoad:",
			@"bk_didFinishWithErrorBlock": @"webView:didFailLoadWithError:"
}];
```
在经过一步代理协议的获取后，进入：

```C
+ (void)bk_linkProtocol:(Protocol *)protocol methods:(NSDictionary *)dictionary
{
    //propertyName为新增的属性block名，selectorName对应于原有的delegate方法名
    //目的是为所有属性生成getter和setter
	[dictionary enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *selectorName, BOOL *stop) {
		// 省略参数检查
		
        // property_copyAttributeValue()第二个参数表示属性的类型
        // D: @dynamic N: nonatomic
        // 以下就是验证属性的存储方式是否符合要求，即动态生成存取方法和Copy赋值对象。详见参考资料Declared Properties。省略实现代码
        
		SEL selector = NSSelectorFromString(selectorName);
		SEL getter = getterForProperty(property, name);
		SEL setter = setterForProperty(property, name);
        
        //使用分类添加的属性自然不能响应getter和setter
		if (class_respondsToSelector(self, setter) || class_respondsToSelector(self, getter)) {
            return;
        }

        //获取上一步注册的代理信息，对于UIWebView来说这里能够获取info
		const A2BlockDelegateInfo *info = [self bk_delegateInfoForProtocol:protocol];

        // 定义属性getter的实现，当在使用时显示访问才会调用。此处省略代码实现。

        // **重要**：调用setter赋值时，进入实现体内部。在这里实现了动态代理的赋值。
		IMP setterImplementation = imp_implementationWithBlock(^(NSObject *delegatingObject, id block) {
			A2DynamicDelegate *delegate = getDynamicDelegate(delegatingObject, protocol, info, YES);
            // 生成对应selector的执行block
			[delegate implementMethod:selector withBlock:block];
		});

        // 关于添加setter是否成功的判断，正常情况下均能成功。省略...
}
```
上面这段代码省略了很多是为了突出核心部分，也就是为属性`setter`的实现，第一句代码内部是这样的：

```C
// 内联函数，static表明是内部函数，不允许外部调用
static inline A2DynamicDelegate *getDynamicDelegate(NSObject *delegatingObject, Protocol *protocol, const A2BlockDelegateInfo *info, BOOL ensuring) {
    // 对于protocol：UIWebViewDelegate， info有效
	A2DynamicDelegate *dynamicDelegate = [delegatingObject bk_dynamicDelegateForProtocol:a2_protocolForDelegatingObject(delegatingObject, protocol)];
  // 后面贴出 
}
// 调用方法实现依次贴出，方便阅读
- (id)bk_dynamicDelegateForProtocol:(Protocol *)protocol {
    // self是将要被代理的对象
	Class class = [A2DynamicDelegate class];
	NSString *protocolName = NSStringFromProtocol(protocol);
	if ([protocolName hasSuffix:@"Delegate"]) {
		// 根据协议名，生成动态代理对象所属类。对于UIWebView实例，它的类就是A2DynamicUIWebViewDelegate。实现细节，可自行查看。
		class = a2_dynamicDelegateClass([self class], @"Delegate");
	} else if ([protocolName hasSuffix:@"DataSource"]) {
		class = a2_dynamicDelegateClass([self class], @"DataSource");
	}

	// 返回一个动态代理实例
	return [self bk_dynamicDelegateWithClass:class forProtocol:protocol];
}

// 懒加载
- (id)bk_dynamicDelegateWithClass:(Class)cls forProtocol:(Protocol *)protocol {
	__block A2DynamicDelegate *dynamicDelegate;

	dispatch_sync(a2_backgroundQueue(), ^{
		dynamicDelegate = objc_getAssociatedObject(self, (__bridge const void *)protocol);

		if (!dynamicDelegate)
		{
            // 动态代理绑定协议信息，例如cls = A2DynamicUIWebViewDelegate	
			dynamicDelegate = [[cls alloc] initWithProtocol:protocol];
            // 给代理对象指定协议绑定动态代理对象
			objc_setAssociatedObject(self, (__bridge const void *)protocol, dynamicDelegate, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
		}
	});

	return dynamicDelegate;
}

```
 以上就获取到了动态代理对象，下面就是将其关联：

```	C
	// 续内联静态方法

    // 以下两个判断的意义是保证info各项参数的完整，防止接下来的消息分发出错。（防御式编程）
	if (!info || !info->setter || !info->getter) {
		return dynamicDelegate;
	}

	if (!info->a2_setter && !info->setter) { return dynamicDelegate; }

    // 两个参数是方法的隐式参数
	id (*getterDispatch)(id, SEL) = (id (*)(id, SEL)) objc_msgSend;
   
    // 获取原始的代理对象，第一次进入自然为空
	id originalDelegate = getterDispatch(delegatingObject, info->getter);

    // 第一次自然条件不会满足，而后就直接返回取得的动态代理对象
	if (bk_object_isKindOfClass(originalDelegate, A2DynamicDelegate.class)) { return dynamicDelegate; }

    // 以下只在第一次设置动态代理对象时调用
	void (*setterDispatch)(id, SEL, id) = (void (*)(id, SEL, id)) objc_msgSend;
    
    // **重要**
    
    // info->a2_setter ?:info->setter 是info->a2_setter ? info->a2_setter : info->setter偷懒的写法。前者条件成立是相当于是空操作，结果就为三目表达式的结果就为条件判断的结果。于是就调用a2_setter，设置真正的代理。
    // 动态调用setter，在注册动态代理是已将系统的setter实现与a2_setter实现交换了。这里的逻辑就是delegatingObject.delegate = dynamicDelegate
	setterDispatch(delegatingObject, info->a2_setter ?: info->setter, dynamicDelegate);

	return dynamicDelegate;
}
```
终于在这步之后，系统能够获取到实际的代理对象了：	`A2DynamicUIWebViewDelegate`类型的实例。

属性的`setter`实现的第二句就是链接`block`实现与协议代理方法：

```C
- (void)implementMethod:(SEL)selector withBlock:(id)block
{
    // 将代理方法的用block实现，self就是现在的那个动态代理对象
   	BOOL isClassMethod = self.isClassProxy;

	if (!block) {
		[self.invocationsBySelectors bk_removeObjectForSelector:selector];
		return;
	}

    // 获取方法的描述：名字及类型。第三个参数指明该方法是否是协议中的必须方法
    // 在必须方法中没找到就在可选方法中找
	struct objc_method_description methodDescription = protocol_getMethodDescription(self.protocol, selector, YES, !isClassMethod);
	if (!methodDescription.name) methodDescription = protocol_getMethodDescription(self.protocol, selector, NO, !isClassMethod);

	A2BlockInvocation *inv = nil;
	if (methodDescription.name) {
		NSMethodSignature *protoSig = [NSMethodSignature signatureWithObjCTypes:methodDescription.types];
        // 使用自定义的block和协议方法签名初始化block调用者。这个构造其中还将验证函数签名和block签名是否一致，不一致将抛出异常，保证block能够正常接收参数并完成回调。详细验证过程请查看源码。
		inv = [[A2BlockInvocation alloc] initWithBlock:block methodSignature:protoSig];
	} else {
        // 协议中没有这个方法，就用block的签名生成函数签名，然后实例化调用对象并返回。
		inv = [[A2BlockInvocation alloc] initWithBlock:block];
	}

    // 以selector为键存储实际调用对象到动态代理的映射表中。
	[self.invocationsBySelectors bk_setObject:inv forSelector:selector];
}

```

```C
- (instancetype)initWithBlock:(id)block {
	NSParameterAssert(block);
	NSMethodSignature *blockSignature = [[self class] typeSignatureForBlock:block];
	// 后面贴出
}
```
这里需要获取到`block`的签名，它与方法不同的是没有`self`和`_cmd`这两个默认参数。作者定义了一个`BKBlockRef`结构体指针来反映其在内存中的数据结构，这里面各个分量可在运行时源码中找到：

```C
typedef struct _BKBlock {
	__unused Class isa;
	BKBlockFlags flags;
	__unused int reserved;
	void (__unused *invoke)(struct _BKBlock *block, ...);
	struct {
		unsigned long int reserved;
		unsigned long int size;
		// requires BKBlockFlagsHasCopyDisposeHelpers
		void (*copy)(void *dst, const void *src);
		void (*dispose)(const void *);
		// requires BKBlockFlagsHasSignature
		const char *signature;
		const char *layout;
	} *descriptor;
	// imported variables
} *BKBlockRef;
```
然后使用指针偏移，获取`signature`：

```C
+ (NSMethodSignature *)typeSignatureForBlock:(id)block __attribute__((pure, nonnull(1))) {
    BKBlockRef layout = (__bridge void *)block;
	// 验证block的标识：签名和Copy赋值的block对象
	if (!(layout->flags & BKBlockFlagsHasSignature))
		return nil;

	void *desc = layout->descriptor;
	desc += 2 * sizeof(unsigned long int);

	if (layout->flags & BKBlockFlagsHasCopyDisposeHelpers)
		desc += 2 * sizeof(void *);

	if (!desc)
		return nil;
   
	const char *signature = (*(const char **)desc);

	return [NSMethodSignature signatureWithObjCTypes:signature];
}
```

```C
	// 接initWithBlock:
	// 将block签名转换成方法签名
	NSMethodSignature *methodSignature = [[self class] methodSignatureForBlockSignature:blockSignature];
	NSAssert(methodSignature, @"Incompatible block: %@", block);
	// 实例化调用对象
	return (self = [self initWithBlock:block methodSignature:methodSignature blockSignature:blockSignature]);
}
```
现在动态代理对象就完成了自定义`block`与代理方法的链接，当代理对象响应代理方法时，就会回调`block`。

在`UIWebView`的分类源文件中，可以看到关于它的代理所属类的定义：

```C
@interface A2DynamicUIWebViewDelegate : A2DynamicDelegate <UIWebViewDelegate>
@end

@implementation A2DynamicUIWebViewDelegate

// UIWebViewDelegate协议方法的实现

@end
```
那么现在我们可以总结下整个动态代理流程了：

 1. 拌合`delegate`的setter方法；
 2. 动态生成属性的存取方法，在`delegate`的`setter`中设置动态代理对象，并将代理方法映射为指定的`block`实现；
 3. 系统获取对象的代理是实际上取得的是动态代理对象（不手动设置代理对象），这个对象遵循了相关协议并实现了协议方法；
 4. 在协议方法实现体中，回调`block`以及`readDelegate`的代理方法（如果有这个对象且其能够响应代理方法）。

例如`webView:shouldStartLoadWithRequest:navigationType:`方法实现：

```C
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
	BOOL ret = YES;
	
	// 如果你指定了代理，代理实现的协议方法和block都将会被调用
	id realDelegate = self.realDelegate;
	if (realDelegate && [realDelegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)])
		ret = [realDelegate webView:webView shouldStartLoadWithRequest:request navigationType:navigationType];

	// 这个方法返回的就是以selector为键的调用者对象的需要回调的block
	BOOL (^block)(UIWebView *, NSURLRequest *, UIWebViewNavigationType) = [self blockImplementationForMethod:_cmd];
	if (block)
		// 判断block与方法签名是否相同的原因
		ret &= block(webView, request, navigationType);

	return ret;
}
```

<font color=330C0580 size=3>- - 这伙计还能个性化定制 - -</font>

目前就`UIWebView`的动态代理的整个流程大体上走了一遍了，你可以跟着以上的分析，结合实例，运用断点，`try again!`然而这还不是终点，你自定义的代理协议也能完美的工作。这里你有两种选择：一是自定义对应的代理对象的实现类，命名方式为：`A2Dynamic类名Delegate`，并动态注册代理和链接代理方法，也就是完全走上面的这条路。相信你不会这么做，于是乎，作者给了我们另外一条路：

```C
// 以下是测试类的接口和实现
#import <Foundation/Foundation.h>

@protocol CustomeDelegate <NSObject>

- (void)scanMe;

- (void)giveMe:(id)any;

@end

@interface CustomeDelegate : NSObject

@property (weak, nonatomic) id<CustomeDelegate> delegate;

- (void)delegateTrigger;

@end
```

```C
import "CustomeDelegate.h"

@implementation CustomeDelegate

- (void)delegateTrigger {
    if ([self.delegate respondsToSelector:@selector(scanMe)]) {
        [self.delegate scanMe];
    }
    
    if ([self.delegate respondsToSelector:@selector(giveMe:)]) {
        [self.delegate giveMe:@"给你"];
    }
}

@end

```

``` C
// 测试实现代码
_del = [[CustomeDelegate alloc] init];
    
// 声明该对象已经实现了代理协议，去掉警告
A2DynamicDelegate<CustomeDelegate> *dd = [_del bk_dynamicDelegateForProtocol:@protocol(CustomeDelegate)];
    
[dd implementMethod:@selector(scanMe) withBlock:^ {
	NSLog(@"scan me");
}];
    
[dd implementMethod:@selector(giveMe:) withBlock:^(id obj) {
	NSLog(@"obj:%@", obj);
}];

// 可见，系统能够知道代理对象是谁，剩下的工作就是如何在接收到代理消息时做出正确的响应。
_del.delegate = dd;

[_del delegateTrigger];
```
如果你注意到`A2DynamicDelegate`是继承自`NSProxy`，而且它本身就是根类。正如它的名字`代理`，它的本质工作就是消息的转发：`Normal forwarding`。为此它必须实现两个父类方法：

```C
- (void)forwardInvocation:(NSInvocation *)invocation;
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel 
```
到这里你可能就有些明悟了，当接收到代理消息时，就把消息转发给动态代理对象，然后它再根据具体的消息回调不同的`block`。而事实上就是这样的过程，请继续往下看！

最开始我们需要通过代理协议对象创建一个动态代理对象，因为我们没有自定义实现类，最终它就是一个`A2DynamicDelegate`实例。然后用指定`block`实现指定的代理方法，这些实现前面已经贴出，下面重点内容是消息转发：

```C
// 返回方法的签名，并生成一个调用实例进入下面的方法
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
	A2BlockInvocation *invocation = nil;
	if ((invocation = [self.invocationsBySelectors bk_objectForSelector:aSelector]))
		return invocation.methodSignature;
	else if ([self.realDelegate methodSignatureForSelector:aSelector])
		return [self.realDelegate methodSignatureForSelector:aSelector];
	else if (class_respondsToSelector(object_getClass(self), aSelector))
		return [object_getClass(self) methodSignatureForSelector:aSelector];
	return [[NSObject class] methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)outerInv
{
	SEL selector = outerInv.selector;
	A2BlockInvocation *innerInv = nil;
	if ((innerInv = [self.invocationsBySelectors bk_objectForSelector:selector])) {
		// 通过调用信息发起调用，调用细节可查看源码。大致分为方法签名验证、调用参数设置。
		[innerInv invokeWithInvocation:outerInv];
	} else if ([self.realDelegate respondsToSelector:selector]) {
		// 如果有指定代理，就把调用信息转发给它
		[outerInv invokeWithTarget:self.realDelegate];
	}
}
```

## 总结
花了近一周的时间，磕磕绊绊算是把这篇文章写完了。最初写这篇文章的动力来自看别人分析这个框架没怎么看懂的郁闷，抱着发挥主观能动性的理念，来自我理解框架的架构思想。不过这个框架大致有哪些功能，还是在那篇文章中知道的，因为之前没用过这个框架，文章链接会在参考资料中给出。

解读这个框架所花时间不可谓长也不可谓短，个人感觉在那茅塞顿开的一瞬间会有所感触：一切都值了。一个成熟的框架涉及到一些底层知识是必不可少的，像是这里面的`block`内存中的数据结构之类的。早就听人言，阅读框架的代码不是要你每行每句都能够吃透，是希望你在这过程中开阔你的编程思想，提供你分析和理解能力。所以文章中一些省略的地方，希望各位能够按照自己的需求去阅读它，毕竟只有自己亲眼看到的才是最为真实的。

最后，发表一点关于这个框架的想法：它本质上是为了让我们能够更为简单的使用API，即用主观上更少的代码完成某项功能，但是背后就要付出更多的代价来支撑省下来的体力活。而我们应当在性能上投注更多的目光，而编写代码过程繁琐与否编程语言本身就已经决定了，要相信所有存在的东西都是有它的道理的。所以这个`BlocksKit`不太推荐使用，但是学习它是完全有必要和有意义的！

## 参考资料
Draveness/analyze: 神奇的BlocksKit[一](https://github.com/Draveness/analyze/blob/master/contents/BlocksKit/神奇的%20BlocksKit%20（一）.md)、[二](https://github.com/Draveness/analyze/blob/master/contents/BlocksKit/神奇的%20BlocksKit%20（二）.md)

Apple Documents: [Declared Properties](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1)

## 勘误
框架作者在`+load`方法中链接的代理方法被`@autoreleasepool`包裹着的目的更切确来说是<font color=#129546>为了防止内存泄漏</font>，因为这个方法是发生在`main()`函数之前，所以自然这其中产生的对象没有被主函数里面的自动释放池管理到。而通过对源码的分析来看，<font color=#129546>其实在`+load`方法的执行过程中前后是加了 `objc_autoreleasePoolPush()` 和` objc_autoreleasePoolPop() 的！`</font> 源码如下：

```C
void call_load_methods(void) {
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}

```
以上在`sunnyxx`的[iOS程序main函数之前发生了什么]( http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)一文中得到印证。
