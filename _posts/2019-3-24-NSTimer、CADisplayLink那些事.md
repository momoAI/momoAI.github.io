>NSTimer计时器对象，就是一个能在从现在开始的后面的某一个时刻或者周期性的执行我们指定的方法的对象。
CADisplayLink也是一个计时器，它的timeInterval和屏幕刷新频率一致（60帧/秒）。

CADisplayLink和NSTimer类似，接下来只着重讲解NSTimer；
#### 创建NSTimer实例
创建NSTimer实例的方法共有8种，其中实例方法2个，类方法6个。这8种方法基本都大同小异，大致可以分为两类，现分别以两个常用方法说明两者区别：
```
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;
```
我们分别用这两个方法创建实例，看下轮询结果：
```
- (void)test {
    NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(testTimer) userInfo:nil repeats:NO];
}

- (void)testTimer {
    NSLog(@"%s",__func__);
}
```
输出结果：[ViewController testTimer]

```
- (void)test {
    NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(testTimer) userInfo:nil repeats:NO];
    
}

- (void)testTimer {
    NSLog(@"%s",__func__);
}
```
无结果，testTimer未调用

区别很明显了（其他scheduled，timerWith开头的方法也是一样）：scheduledTimerWithTimeInterval方法创建实例后执行了轮询的方法，这是因为这个方法创建的timer会被自动添加到runloop中，由runloop处理轮询事件。而timerWithTimeInterval方法创建timer后，需我们手动添加至runloop中；
CADisplayLink只有一个实例化方法，同样道理，也需要手动添加至runloop中：
```
+ (CADisplayLink *)displayLinkWithTarget:(id)target selector:(SEL)sel;
- (void)addToRunLoop:(NSRunLoop *)runloop forMode:(NSRunLoopMode)mode;
```
scheduledTimerWithTimeInterval方法虽能自动添加至runloop中，但这个runloop只会是当前runloop，runloop的模式也只是NSDefaultRunLoopMode；而手动添加至runloop，则可以自己设置指定的runloop及runloopMode；我们改写第二段代码看下：
```
- (void)test {
    NSTimer *timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(testTimer) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
}
```
>[ViewController testTimer]

那timer为什么一定要添加至runloop中才会执行呢？

#### NSTimer与NSRunLoop
在iOS中，所有消息都会被添加到NSRunloop中，在NSRunloop中分为‘input source’跟'timer source'，并在循环中检查是不是有事件需要发生，如果需要那么就调用相应的函数处理。NSTimer这种'timer source'资源要想起作用，那肯定需要加到runloop中才会执行了。
将NSTimer添加至runloop中，也需指定runloop mode；runloop mode有很多种，接下来我们主要讲下以下3种：

- NSDefaultRunLoopMode
- UITrackingRunLoopMode
- NSRunLoopCommonModes

`NSDefaultRunLoopMode：`这个是默认的运行模式，也是最常用的运行模式，NSTimer与NSURLConnection默认运行该模式下。默认模式中几乎包含了所有输入源(NSConnection除外)，一般情况下应使用此模式。
`UITrackingRunLoopMode：`当滑动屏幕的时候，比如UIScrollView或者它的子类UITableView、UICollectionView等滑动时runloop处于这个模式下。在此模式下会限制输入事件的处理。
`NSRunLoopCommonModes：`这是一组可配置的常用模式，是一个伪模式，其为一组runloop mode的集合。将输入源与此模式相关联还将其与组中的每种模式相关联，当输入源加入此模式意味着在CommonModes中包含的所有模式下都可以处理。 对于iOS应用程序，默认情况下此设置包括NSDefaultRunLoopMode和UITrackingRunLoopMode。可使用`CFRunLoopAddCommonMode`方法在CommonModes中添加自定义modes。

了解这几种runloop modes后，我们就知道平时开发中timer在屏幕滑动的时候停止执行的原因了，也能轻易的解决这个问题：
一个timer可以被添加到runloop的多个模式，因此如果想让timer在UIScrollView等滑动的时候也能够触发，就可以分别添加到UITrackingRunLoopMode，UITrackingRunLoopMode这两个模式下。或者直接添加到NSRunLoopCommonModes这一个模式集，它包含了上面的两种模式。

另外，一个线程有且只有一个runloop。主线程的runloop默认是开启的，其他线程runloop并不是默认开启的。所以当在子线程中添加timer时，还需手动开启这个runloop：
```
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        NSTimer *timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(testTimer) userInfo:nil repeats:NO];
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        [[NSRunLoop currentRunLoop] run];
    });
```
#### NSTimer触发事件的准时性
NSTimer不是一个实时系统，不管是一次性的还是周期性的timer的实际触发事件的时间可能都会跟我们预想的会有出入。NSTimer并不是准时的，造成不准时的原因如下：
- 触发点可能被跳过去
  1. 线程处理比较耗时的事情时会发生这种情况，当线程处理的事情比较耗时时，timer就会延迟到该事件执行完以后才会执行。
  2. timer添加到的runloop mode不是runloop当前运行的mode，也会发生这种情况。

  虽然跳过去，但是接下来的执行不会依据被延迟的时间加上间隔时间，而是根据之前的时间来执行。比如：
定时时间间隔为2秒，t1秒添加成功，那么会在t2、t4、t6、t8、t10秒注册好事件，并在这些时间触发。假设第3秒时，执行了一个超时操作耗费了5.5秒，则触发时间是：t2、t8.5、t10，第4和第6秒就被跳过去了，虽然在t8.5秒触发了一次，但是下一次触发时间是t10，而不是t10.5。
- 不准点
  比如上面说的t2、t4、t6、t8、t10，并不会在准确的时间触发，而是会延迟个很小的时间。也有两个因素：
  1. RunLoop为了节省资源，并不会在非常准确的时间点触发。
  2. 线程有耗时操作，或者其它线程有耗时操作也会影响

  好在，这种不准点对我们开发影响并不大（基本是毫秒级别以下的延迟）

#### NSTimer内存问题
重点来了！！！
还是先看段代码
```
- (void)test {
    self.timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(testTimer) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
}

- (void)dealloc {
    NSLog(@"%s",__func__);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%s",__func__);
    [self dismissViewControllerAnimated:YES completion:nil];
}
```
点击屏幕，输出结果：
[TestViewController touchesBegan:withEvent:]
[TestViewController dealloc]
接下来，我们将创建timer时的repeats改为YES再看下：
```
self.timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(testTimer) userInfo:nil repeats:YES];
```
点击屏幕，输出结果：
[TestViewController touchesBegan:withEvent:]
可以看出，repeats改为YES后TestViewController没有被释放，发生了内存泄露。为什么会这样呢？
前面说了，NSTimer是要添加至runloop中的，runloop会对timer有强引用。而timer会对目标对象target进行强引用（类似UIButton等并不会强引用target）。如果viewController也强引用了timer，那viewController和timer循环引用了。而就算viewController没有强引用timer，由于runloop对timer的强引用一直存在，timer又强引用着viewController，viewController也不能释放。它们的引用关系如下：

![haha](https://upload-images.jianshu.io/upload_images/2427856-5182442b57a01b0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于单次事件的timer来说，在事件执行完后，timer会把本身从NSRunLoop中移除，然后就是释放对‘target’对象的强引用，所以不会有内存泄露问题。NSTimer的- (void)invalidate方法就是这个作用，所以对于重复性的timer在不需要的时候调用invalidate方法也能解决内存问题：
```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%s",__func__);
    [_timer invalidate];
    [self dismissViewControllerAnimated:YES completion:nil];
}
```
点击屏幕，输出结果：
[TestViewController touchesBegan:withEvent:]
[TestViewController dealloc]

But,
调用invalidate方法，这是在我们能明确timer销毁的时机的基础上调用的（就像上面的代码，我们能明确的知道touchesBegan事件时停止timer）。但大部分时候的需求并没有明确的时机，而是让timer一直执行直到所在VC销毁时才停止timer，也就是说我们必须在dealloc方法中调用timer的invalidate方法以确保万无一失。但timer未调用invalidate方法VC并不会销毁，不会调用dealloc方法；dealloc没调用，那timer的invalidate调用不了。这就尴尬了，陷入了死结。这该如何是好？
这种情况，其实有多种解决方案。最简单的方案就是：在viewWillAppear方法中创建、执行timer，在viewWillDisappear中销毁timer。这种方式虽然能解决内存问题，但实现过程比较low会有潜在的bug。其实也算不上解决方案，不推荐使用。
下面，介绍比较高大上比较有技术含量，并且能完美解决内存泄漏问题的方案；
首先我们分析下解决timer造成内存泄漏的突破点：
我们再看下上面的内存引用图。runloop对timer的强引用是一直存在的，我们无法改变这种状态（除非timer调用invalidate方法）；前面也分析了：当runloop强引用timer时，viewController对timer是否是强引用viewController都释放不了。那么唯一的突破点就是打破timer对target（即viewController）的强引用，让viewController能正常释放，然后调用dealloc方法进而销毁timer将timer本身从NSRunLoop中移除。如何能做到呢？
这有两种方案：
- 创建NSTimer分类，将target设置为本身的NSTimer类对象，将selector的方式改为block回调方式；
```
@interface NSTimer (JSBlockSupport)

+ (NSTimer *)js_scheduledTimerWithTimeInterval:(NSTimeInterval)ti block:(void(^)(void))block repeats:(BOOL)yesOrNo;

@end

@implementation NSTimer (JSBlockSupport)

+ (NSTimer *)js_scheduledTimerWithTimeInterval:(NSTimeInterval)ti block:(void (^)(void))block repeats:(BOOL)yesOrNo {
    return [self scheduledTimerWithTimeInterval:ti target:self selector:@selector(js_blockInvoke:) userInfo:[block copy] repeats:yesOrNo];
}

+ (void)js_blockInvoke:(NSTimer *)timer {
    void (^block)(void) = timer.userInfo;
    if (block) {
        block();
    }
}

@end
```

```
- (void)test {
    __weak typeof (self) weakSelf = self;
    self.timer = [NSTimer js_scheduledTimerWithTimeInterval:1 block:^{
        __strong typeof (weakSelf) strongSelf = weakSelf;
        [strongSelf testTimer];
    } repeats:YES];
}

- (void)dealloc {
    NSLog(@"%s",__func__);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%s",__func__);
//    [_timer invalidate];
    [self dismissViewControllerAnimated:YES completion:nil];
}
```
点击屏幕，输出结果：
[TestViewController touchesBegan:withEvent:]
[TestViewController dealloc]
这里很巧妙的利用了timer的userInfo属性，将执行的操作作为block赋值给userInfo（id类型）,在分类中的selector取出userInfo并执行这个block。
在调用分类方法时，有个注意点：block里需使用weak对象，再用strong对象调用方法。这是因为viewController强引用timer，timer强引用block，block会捕获里面的viewController，这同样循环引用了，本来我们是为了处理内存问题的，结果一不小心就全白忙了。用strong对象调用方法，是为了保证对象一直存活执行方法。

我们再回过头来看下CADisplayLink如何处理，由于CADisplayLink没有类似userInfo这样的属性，所以我们需要为CADisplayLink分类添加属性，为分类添加属性就要用到runtime关联属性了：
```
@interface CADisplayLink (JSBlockSupport)

+ (CADisplayLink *)js_displayLinkWithBlock:(void(^)(void))block;

@end

static void *JSDisplayLinkKey = "JSDisplayLinkKey";

@implementation CADisplayLink (JSBlockSupport)

+ (CADisplayLink *)js_displayLinkWithBlock:(void (^)(void))block {
    CADisplayLink *displayLink = [self displayLinkWithTarget:self selector:@selector(js_blockInvoke:)];
    // 直接关联了一个block对象
    objc_setAssociatedObject(displayLink, JSDisplayLinkKey, block, OBJC_ASSOCIATION_COPY);
    return displayLink;
}

+ (void)js_blockInvoke:(CADisplayLink *)displayLink {
    void (^block)(void) = objc_getAssociatedObject(displayLink, JSDisplayLinkKey);
    if (block) {
        block();
    }
}

@end

```

```
- (void)test {
    __weak typeof (self) weakSelf = self;
    self.displayLink = [CADisplayLink js_displayLinkWithBlock:^{
        __strong typeof (weakSelf) strongSelf = weakSelf;
        [strongSelf testTimer];
    }];
    [self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
}
```
- 为timer原来的target设置NSProxy代理对象，将这个代理设置为timer的target。这样timer没有强引用原本target对象了，而代理对象收到的消息通过消息转发机制又转发给原本target处理。
```
- (void)test {
    self.timer = [NSTimer timerWithTimeInterval:1 target:[YYWeakProxy proxyWithTarget:self] selector:@selector(testTimer) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
}
```
点击屏幕，输出结果：
[TestViewController touchesBegan:withEvent:]
[TestViewController dealloc]
这里创建NSProxy代理对象，使用了大神写的YYWeakProxy类，具体实现可以下载看下其源码。

- iOS10添加了block方式的初始化方法，同样也解决了内存泄漏问题（但一般项目都要适配iOS10之前的系统，所以也并没有什么用处）：
```
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));
```
