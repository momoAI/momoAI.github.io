首先，我们看下优秀的自动布局第三方框架Masonry/SnapKit的链式语法应用：
```
      // Masonry oc
     [box mas_makeConstraints:^(MASConstraintMaker *make) {
          make.width.height.equalTo(@50);
          make.center.equalTo(self.view);
      }];

```
```
        // SnapKit  swift
        box.snp.makeConstraints { (make) -> Void in
           make.width.height.equalTo(50)
           make.center.equalTo(self.view)
        }
```
Objective-C、Swift版都使用了连续的.语法，这就是链式编程的特点。链式编程就是将多个方法用点语法链接起来，让代码更加简洁，可读性更强。
附一篇讲解Masonry源码的博文：[iOS开发之Masonry框架源码解析](https://www.cnblogs.com/ludashi/p/5591572.html)
###那么用Objective-C怎么实现链式编程呢？
以Masonry那段代码为例，连续调用中有width，height等不带（）参数的，也有equalTo(@50)这种带参数的。我们知道OC是通过[ ]语法调用方法的，能使用点语法的只能是类的属性了（getter setter方法）。
- 对于不带参数的，直接通过点语法调用getter方法就能实现。但getter方法返回的是属性本身，而连续的点语法需要返回调用的对象以此才能调用下个点语法。一种做法是将属性的类型声明为调用对象类型，但这样做耦合度太高。因此要使用另一种方式，这需要用到代理：A对象为链式调用者，B对象作为A的属性，A设为B的代理且B中定义和A一样的属性名。这样即使调用B的getter方法返回的是B对象，再调用下个点语法时B通过代理又调用了A的方法。
- 对于带参数的，一般属性类似于self.title通过点语法访问，后面是不带()的。看到()我们很容易联想到block，block是通过()调用的。所以这个属性一定是block，block可以设置参数。而要实现连续点语法，这个block一定要有返回值，且返回值是调用block的对象。整个思路就出来了（不带参数的也同样能用block实现）

通过demo实现链式编程简单的计算器功能，方便直观的理解这种写法：
##### 代理方式：
```
#import <Foundation/Foundation.h>

@class Mediator;

@protocol MediatorDelegate<NSObject>

- (Mediator *)add;
- (Mediator *)sub;
- (float)result;

@end


@interface Mediator : NSObject

@property (nonatomic, strong) Mediator *add;
@property (nonatomic, strong) Mediator *sub;
@property (nonatomic, assign) float result;
@property (nonatomic, weak) id<MediatorDelegate> delegate;

@end
```
```
#import "Mediator.h"

@implementation Mediator

- (Mediator *)add {
    return [self.delegate add];
}

- (Mediator *)sub {
    return [self.delegate sub];
}

- (float)result {
    return [self.delegate result];
}

@end
```
```
#import <Foundation/Foundation.h>
#import "Mediator.h"

@interface Calculator : NSObject<MediatorDelegate>

@property (nonatomic, assign) float result; // 存储计算结果
@property (nonatomic, strong) Mediator *add;
@property (nonatomic, strong) Mediator *sub;

@end
```
```
#import "Calculator.h"

@implementation Calculator

- (Mediator *)add {
    if (!_add) {
        _add = [[Mediator alloc] init];
        _add.delegate = self;
    }
    _result ++;
    return _add;
}

- (Mediator *)sub {
    if (!_sub) {
        _sub = [[Mediator alloc] init];
        _sub.delegate = self;
    }
    _result --;
    return _sub;
}

- (float)result {
    return _result;
}

@end
```
```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    Calculator *calculator = [[Calculator alloc] init];
    float result = calculator.add.sub.add.result; // 1
}
@end
```

##### block方式：
```
#import <Foundation/Foundation.h>

@class Calculator;

typedef Calculator *(^CalculatorBlock)(float value);

@interface Calculator : NSObject

@property (nonatomic, assign) float result; // 存储计算的结果
@property (nonatomic, copy) CalculatorBlock add;
@property (nonatomic, copy) CalculatorBlock sub;

@end
```
```
#import "Calculator.h"

@implementation Calculator

- (CalculatorBlock)add {
    return ^Calculator *(float value) {
        self.result += value;
        return self;
    };
}

- (CalculatorBlock)sub {
    return ^Calculator *(float value) {
        self.result -= value;
        return self;
    };
}

@end
```
计算器Calculator类定义了加法add和减法sub两个block属性，这个block有一个float参数，返回值为Calculator对象。然后通过重写getter方法实现自己的计算，并用属性result记录。因为block返回的是self，在调用第一个block后仍可以调用下个block，点语法的链条就形成了。
现在我们就可以通过链式调用计算器加减法了：
```
#import "ViewController.h"
#import "Calculator.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    Calculator *calculator = [[Calculator alloc] init];
    float result = calculator.add(1).sub(2).add(3).result; // 2
}

@end
```
为了更清楚理解这种写法，我把calculator.add(1).sub(2).add(3).result;这句代码过程再逐个拆解下：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    Calculator *calculator = [[Calculator alloc] init];
//    float result = calculator.add(1).sub(2).add(3).result;
    
    CalculatorBlock block1 = calculator.add;
    calculator = block1(1);
    
    CalculatorBlock block2 = calculator.sub;
    calculator = block2(2);
    
    CalculatorBlock block3 = calculator.add;
    calculator = block3(3);
    
    float result = calculator.result; // 2
}
```
#### swift语言的链式编程
大部分面向对象语言都是可以通过点语法调用方法的，这些语言写链式代码就不会像OC这么麻烦了。这里再用swift语法写刚才那个计算器，对比一下：
```
import UIKit

class Calculator: NSObject {
    public var result:Float = 0;
    
    public func add(_ value:Float) -> Calculator {
        result += value;
        return self;
    }
    
    public func sub(_ value:Float) -> Calculator {
        result -= value;
        return self;
    }
}
```
```
import UIKit

class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let calculator = Calculator()
        let result = calculator.add(1).sub(2).add(3).result // 2
    }
}
```
