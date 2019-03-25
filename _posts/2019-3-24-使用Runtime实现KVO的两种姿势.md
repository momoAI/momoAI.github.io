**KVO**
>Key-Value Observing，KVO 是一种观察者模式的实现：当被观察对象的某个属性发生更改时，观察者对象会获得通知。

KVO使用很简单：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    self.text = @"1";
    // 添加observer
    [self addObserver:self forKeyPath:@"text" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];
}
// key值改变
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSString *text = [NSString stringWithFormat:@"%i",arc4random()%100];
    NSLog(@"text change to %@",text);
    self.text = text;
}
// 监听
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSString *oldValue = [change objectForKey:NSKeyValueChangeOldKey];
    NSString *newValue = [change objectForKey:NSKeyValueChangeNewKey];
    NSLog(@"observer oldValue:%@=====>newValue:%@",oldValue,newValue);
}
```
```
2018-11-11 21:52:31.832483+0800 FWCustomKVO[863:22943] origin setter:1
2018-11-11 21:52:43.410054+0800 FWCustomKVO[863:22943] text change to 8
2018-11-11 21:52:43.410308+0800 FWCustomKVO[863:22943] origin setter:8
2018-11-11 21:52:43.410476+0800 FWCustomKVO[863:22943] observer oldValue:1=====>newValue:8
```
可以看到，不需要给被观察的对象添加任何额外代码，只是简单的`-observeValueForKeyPath:ofObject:change:context: `就能使用 KVO 。这是怎么做到的呢？

#### KVO 实现原理
KVO 的实现使用了 Runtime：当你观察一个对象时，一个新的类会被动态创建。这个类继承自该对象的原本的类，并重写了key属性的 setter 方法。通过重写 setter 方法会在调用原 setter 方法之前和之后，通知所有观察对象值的更改。最后把这个对象的 isa 指针指向这个新创建的子类，对象就变成了原本类的子类的实例。这个动态创建的子类还重写了 `- class` 方法返回了原本类的class，我们使用`- class`方法查看被观察到对象类信息时，并不能发现它其实已经改变了。

#### 自己实现KVO
虽然，我们调用2句代码就能使用KVO，但系统的KVO有时并不满足我们的需求。比如，你只能通过重写 
`-observeValueForKeyPath:ofObject:change:context: `方法来获得通知。我们既不能使用自定义的 selector ，也不能使用 block 。而且父类同样监听同一个对象的同一个属性时还要处理父类的情况。
既然我们已经知道了KVO的实现原理，那为何不自己使用Runtime实现KVO呢？我们现在就来实现一个block回调的KVO.
这里有两种实现方式（当然都是使用Runtime），我们先模仿系统KVO的实现：
- **姿势1**
新建NSObject分类，实现类似系统提供的两个方法addObserver和removeObserver：
```
@interface NSObject (KVO)
- (void)fw_addObserver:(NSObject *)observer
                forKey:(NSString *)key
             callBack:(FWObserverBlock)block;

- (void)fw_removeObserver:(NSObject *)observer forKey:(NSString *)key;

@end
```
```
static const void * kFWKVOAssociateKey = "FWKVOAssociateKey";
static NSString * kFWKVOClassPrefix = @"FWKVOClassPrefix_";

@implementation NSObject (KVO)

- (void)fw_addObserver:(NSObject *)observer forKey:(NSString *)key callBack:(FWObserverBlock)block {
    SEL setterSel = NSSelectorFromString(setterForKey(key));
    Class cls = object_getClass(self);
    NSString *className = NSStringFromClass(cls);
    // 类没有更改过
    if (![className hasPrefix:kFWKVOClassPrefix]) {
        // 创建自定义KVO类
        Class cls_kvo = [self makeKVOClassWithOriginClassName:className];
        object_setClass(self, cls_kvo); // isa指向创建的子类
    }
    
    // 更改创建的KVO类的setter方法
    if (![self hasSelector:setterSel]) {
        Method setterMethod = class_getInstanceMethod([self class], setterSel);
        class_addMethod(object_getClass(self), setterSel, (IMP)fw_setter, method_getTypeEncoding(setterMethod));
    }
    
    // 关联对象
    NSMutableArray *observers = objc_getAssociatedObject(self, kFWKVOAssociateKey);
    if (!observers) { // 没有关联
        observers = [NSMutableArray array];
        objc_setAssociatedObject(self, kFWKVOAssociateKey, observers, OBJC_ASSOCIATION_RETAIN);
    }
    FWObserverModel *model = [[FWObserverModel alloc] initWithObserver:observer key:key block:block];
    [observers addObject:model];
}

- (Class)makeKVOClassWithOriginClassName:(NSString *)className {
    NSString *KVOClassName = [kFWKVOClassPrefix stringByAppendingString:className];
    Class cls_kvo = NSClassFromString(KVOClassName);
    if (cls_kvo) {
        return cls_kvo;
    }
    Class cls_base = object_getClass(self);
    // 创建子类
    cls_kvo = objc_allocateClassPair(cls_base, KVOClassName.UTF8String, 0);
    
    // 更改class方法 调用父类的 (迷惑作用)
    Method classMethod = class_getInstanceMethod(cls_base, @selector(class));
    class_addMethod(cls_kvo, @selector(class), method_getImplementation(classMethod), method_getTypeEncoding(classMethod));
    objc_registerClassPair(cls_kvo);
    
    return cls_kvo;
}

void fw_setter(id self,SEL _cmd,id newValue) {
    NSString *setter = NSStringFromSelector(_cmd);
    NSString *getter = getterForKey(setter);
    id oldValue = [self valueForKey:getter]; // kvc获取更改前的值
    // 获取关联对象   block回调
    NSMutableArray *observers = objc_getAssociatedObject(self, kFWKVOAssociateKey);
    for (FWObserverModel *model  in observers) {
        if ([model.key isEqualToString:model.key]) {
            model.block(self, getter, oldValue, newValue);
        }
    }
    
    // 调用setter原始方法 即自定义KVO类的父类方法
    struct objc_super superClass = {
        .receiver = self,
        .super_class = class_getSuperclass(object_getClass(self))
    };
    void (*objc_msgSendSuperCasted)(void *, SEL, id) = (void *)objc_msgSendSuper;
    objc_msgSendSuperCasted(&superClass, _cmd, newValue);
}

- (void)fw_removeObserver:(NSObject *)observer forKey:(NSString *)key {
    NSMutableArray *observers = objc_getAssociatedObject(self, kFWKVOAssociateKey);
    FWObserverModel *removeObserver;
    for (FWObserverModel *object in observers) {
        if ([object.key isEqualToString:key] && object.observer == observer) {
            removeObserver = object;
        }
    }
    [observers removeObject:removeObserver];
}

@end
```
测试一下，自己写的KVO是否生效：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    self.text = @"1";
    [self fw_addObserver:self forKey:@"text" callBack:^(id observer, NSString *key, id oldValue, id newValue) {
        NSLog(@"observer oldValue:%@=====>newValue:%@",oldValue,newValue);
    }];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSString *text = [NSString stringWithFormat:@"%i",arc4random()%100];
    NSLog(@"text change to %@",text);
    self.text = text;
}

- (void)setText:(NSString *)text {
    _text = text;
    NSLog(@"origin setter:%@",text);
}
```
完美：
```
2018-11-11 22:01:40.128786+0800 FWCustomKVO[1202:48617] origin setter:1
2018-11-11 22:01:43.296806+0800 FWCustomKVO[1202:48617] text change to 81
2018-11-11 22:01:43.297037+0800 FWCustomKVO[1202:48617] observer oldValue:1=====>newValue:81
2018-11-11 22:01:43.297148+0800 FWCustomKVO[1202:48617] origin setter:81
```

- **姿势2**
上面的方式动态的创建了子类，接下来介绍的方式可以不用创建类而是使用Method Swizzling黑魔法：通过替换被监听类的属性的setter方法，实现我们自己的功能：
同样，还是需要创建分类，修改fw_addObserver方法实现即可
```
- (void)fw_addObserver:(NSObject *)observer forKey:(NSString *)key callBack:(FWObserverBlock)block {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        SEL originalSel = NSSelectorFromString(setterForKey(key));
        SEL targetSel = NSSelectorFromString(@"origin_setter:");
        Method originalMethod = class_getInstanceMethod([self class], originalSel);
        Method targetMethod = class_getInstanceMethod([self class], targetSel);
        // addMethod判断  因为当前类（或父类）可能已经有该method了
        BOOL addRet = class_addMethod([self class], targetSel, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        if (!addRet) {
            method_setImplementation(targetMethod, method_getImplementation(originalMethod));
        }
        // 替换原来setter方法的IMP
        method_setImplementation(originalMethod, (IMP)fw_setter);
    });

    // 关联对象
    NSMutableArray *observers = objc_getAssociatedObject(self, kFWKVOAssociateKey);
    if (!observers) { // 没有关联
        observers = [NSMutableArray array];
        objc_setAssociatedObject(self, kFWKVOAssociateKey, observers, OBJC_ASSOCIATION_RETAIN);
    }
    FWObserverModel *model = [[FWObserverModel alloc] initWithObserver:observer key:key block:block];
    [observers addObject:model];
}

void fw_setter(id self,SEL _cmd,id newValue) {
    NSString *setter = NSStringFromSelector(_cmd);
    NSString *getter = getterForKey(setter);
    id oldValue = [self valueForKey:getter]; // kvc获取更改前的值
    // 获取关联对象   block回调
    NSMutableArray *observers = objc_getAssociatedObject(self, kFWKVOAssociateKey);
    for (FWObserverModel *model  in observers) {
        if ([model.key isEqualToString:model.key]) {
            model.block(self, getter, oldValue, newValue);
        }
    }

    // 调用setter原始方法
    SEL selector = NSSelectorFromString(@"origin_setter:");
    ((void (*)(id,SEL,id))objc_msgSend)(self,selector,newValue);
}
```
其中`SEL targetSel = NSSelectorFromString(@"origin_setter:");`这个自定义方法并不需要实现，因为我们只是利用这个来记录原始setter方法的实现而已，之后可以通过调用origin_setter:方法实现调用原始setter方法的目的。

另外最后使用了objc_msgSend发送消息的方式调用方法，其实还有两种方式：
```
    // 2.
void (*objc_msgSendCasted)(id,SEL,id) = (void *)objc_msgSend;
objc_msgSendCasted(self,selector,newValue);
```
```
    // 3.
[self performSelector:selector withObject:newValue];
```
