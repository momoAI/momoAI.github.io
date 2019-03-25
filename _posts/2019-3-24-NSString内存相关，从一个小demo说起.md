![timg1.jpg](https://upload-images.jianshu.io/upload_images/2427856-43b29741255cac80.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


首先，我们看下第一段代码：

```
- (void)stringTest {
    NSString *string1 = @"string";
    NSString *string2 = [NSString stringWithString:@"string"];
    NSString *string3 = [[NSString alloc] initWithString:@"string"];
    NSString *string4 = [NSString stringWithFormat:@"string"];
    NSString *string5 = [[NSString alloc] initWithFormat:@"string"];
    NSLog(@"1----%p",string1);
    NSLog(@"2----%p",string2);
    NSLog(@"3----%p",string3);
    NSLog(@"4----%p",string4);
    NSLog(@"5----%p",string5);
}
```

> 2018-04-27 17:36:17.419547+0800 StringTest[19381:2393524] 1----0x10dd070e8
2018-04-27 17:36:17.419789+0800 StringTest[19381:2393524] 2----0x10dd070e8
2018-04-27 17:36:17.420025+0800 StringTest[19381:2393524] 3----0x10dd070e8
2018-04-27 17:36:17.420166+0800 StringTest[19381:2393524] 4----0xa00676e697274736
2018-04-27 17:36:17.420258+0800 StringTest[19381:2393524] 5----0xa00676e697274736

这里分别用了不同方法创建了NSString对象
哪它们到底有什么区别呢？我们先看看存储区域的划分：
# 存储区域
- 栈区（stack） 
　　　由编译器自动分配释放   ，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。  
- 堆区（heap）
　　　一般由程序员分配释放，   若程序员不释放，程序结束时可能由OS回收 。与数据结构中的堆是两回事，分配方式倒是类似于链表。  
- 全局区（静态区）（static）
　　　全局变量和静态变量的存储是放在一块的（全局变量就是采取静态存储方式的），初始化的全局变量和静态变量在一块区域，未初始化的全局变量和未初始化的静态变量在相邻的另 
    一块区域， 程序结束后由系统释放。
- 文字常量区   
　　　常量字符串就是放在这里的，程序结束后由系统释放。  
- 代码区
　　　存放函数体的二进制代码。

![内存分区](https://upload-images.jianshu.io/upload_images/2427856-4d1801f6766bb974.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. string1通过字面量创建，它是一个常量存储在常量区。如果其它对象存储的内容一样，则指针指向相同的地址。不会初始化内存空间，所以使用结束后不会释放内存。
2. string2通过类方法初始化string创建，是通过copy @"string"返回一个字符串，且这个copy是浅拷贝，会指向同一块地址。其实就是相当于字面量创建的方式，完全是多余的，所以这段代码会有警告：Using 'initWithString:' with a literal is redundant。
3. string3通过实例方法初始化string创建，和string2类似:相当于字面量创建，也会有警告。
4. string4通过类方法初始化format创建，需要初始化一段动态内存空间，存储在堆中，使用结束后需释放内存。
5. string5通过实例方法初始化format创建，和string4类似。

initWith...和stringWith...这两种实例方法和类方法的内存分配情况都是一样的，但内存释放却又有区别。什么区别呢，我们继续看第二段代码：

```
@interface ViewController ()

@property (nonatomic, weak) NSString *string1;
@property (nonatomic, weak) NSString *string2;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self stringTest];
    NSLog(@"%@",_string1);
    NSLog(@"%@",_string2);
}

- (void)stringTest {
    NSString *string1 = [NSString stringWithFormat:@"string string1"];
    NSString *string2 = [[NSString alloc] initWithFormat:@"string string2"];
    self.string1 = string1;
    self.string2 = string2;
}

@end
```

这里两个string属性都用了weak修饰而没有使用strong，这样就可以通过打印它们的值分析string1,string2内存释放的情况了。一般情况下，只在需要避免循环引用时使用weak修饰符。runtime 对注册的类， 会进行布局，对于 weak 对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会 dealloc。因此weak修饰的变量所引用对象被废弃时会经过以下步骤：
>  1.从weak表中获取废弃对象的地址为键值的记录。
2.将包含在记录中的weak修饰符变量的地址赋值为nil.
3.从weak表中删除该记录。
4.从引用计数表中删除废弃对象的地址为键值的记录。

这会消耗相应的CPU资源。这里加weak只是为了方便分析。
这段代码的输出结果是什么呢？
由于_string1和_string2都是弱引用，而局部变量string1, string2在调用stringTest方法后就由系统销毁了，你可能会不假思索的回答：结果都为null，但我们看下实际情况：
> 2018-04-28 11:36:20.445170+0800 StringTest[22110:2816244] string string1
2018-04-28 11:36:20.445294+0800 StringTest[22110:2816244] (null)

为什么string1还会有值呢？string1引用的对象在方法作用域外不是应该销毁了吗？其实这里涉及到iOS内存管理的另外一个知识点：自动释放池autoreleasepool。
#autorelease
我们知道autorelease是一种内存自动回收机制，autorelease的对象会被添加到autoreleasepool。autoreleasepool中的对象不会马上release。在正常情况下，创建的对象会在超出其作用域的时候release，但是如果将对象加入autoreleasepool，那么该对象会等到autoreleasepool销毁的时候再释放，这使得对象超出其指定的生存范围时能够自动并正确地释放。通过类似+ (instancetype)stringWithFormat:(NSString *)string方法创建的string1对象，就是被添加到了autoreleasepool中是一个autorelease对象。因为在我们没有手动加autoreleasepool的情况下，autorelease对象要在当前的runloop迭代结束时才废弃的，也就是说string1要在循环结束后才释放（因为runloop每次循环过程中autoreleasepool被生成或废弃）
因此，我们有两种方式改写代码，让string1释放：
第一种方式：手动加@autoreleasepool，在@autoreleasepool {}后对象就会释放了

```
- (void)viewDidLoad {
    [super viewDidLoad];
    @autoreleasepool {
        [self stringTest];
    }
    NSLog(@"%@",_string1);
    NSLog(@"%@",_string2);
}
```

第二种方式：模拟runloop迭代

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self stringTest];
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"%@",_string1);
        NSLog(@"%@",_string2);
    });
}
```

打印结果都是一样的：
> 2018-04-28 14:18:10.769891+0800 StringTest[22742:2893498] (null)
2018-04-28 14:18:10.770081+0800 StringTest[22742:2893498] (null)

现在我们大致知道了，stringWithFormat创建的对象会被添加到自动释放池是自动释放对象，而initWithFormat创建的对象不会被添加到自动释放池。不只是NSString类，其他类都是如此的。ARC下生成的对象不能调用autorelease，ARC下为了区分生成的对象是不是autorelease，就确立了硬性规则。这些规则简单地体现在了方法名上。
使用alloc/new/copy/mutableCopy名称开头的方法意味着生成的对象调用者持有,这些自己生成并持有的对象通过release释放(`额外说下，这也是声明一个new前缀的属性编译会报错的原因，因为这个属性的getter方法以new开头，按照硬性规则内存可能就不对了`)。而其他名称类似string array dictionary的方法生成对象，不归调用者持有，这种情况下，对象是自动释放。也就是说

```
  NSString *string1 = [NSString stringWithFormat:@"string string1"];
```

等同于

```
    NSString *string1 = [[[NSString alloc] initWithFormat:@"string string1"] autorelease];
```

眼尖的小伙伴肯定注意到了，上面的第二段代码中string1和string2初始化的值分别是@"string string1",@"string string2"，这个是我特意这样写的为的是字符串的长度。那这个长度对打印的结果有什么影响呢？把第二段代码改下：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self stringTest];
    NSLog(@"%@",_string1);
    NSLog(@"%@",_string2);
}

- (void)stringTest {
    NSString *string1 = [NSString stringWithFormat:@"string1"];
    NSString *string2 = [[NSString alloc] initWithFormat:@"string2"];
    self.string1 = string1;
    self.string2 = string2;
}
```

> 2018-04-28 16:10:17.572271+0800 StringTest[23270:2961270] string1
2018-04-28 16:10:17.572456+0800 StringTest[23270:2961270] string2

有没有发现，string2也没有释放了。刚才不是说了init创建的对象不会添加到autoreleasepool，怎么改了字符串值就没有释放了呢？这个其实已经与autoreleasepool没有关系了，我们可以试下像之前一样手动加@autoreleasepool但结果并不会改变。这里又涉及到另个一知识点了：
# Tagged Pointer
我们通过lldb看下string1的信息：
> (lldb) p string1
(NSTaggedPointerString *) $1 = 0xa31676e697274737 @"string1"

可以看到NSTaggedPointerString这样奇怪的类，它就是Tagged Pointer对象。那它时干什么的呢？
假设要存储一个NSNumber对象，其值是一个整数。正常情况下，如果这个整数只是一个NSInteger的普通变量，那么它所占用的内存是与CPU的位数有关，在32位CPU下占4个字节，在64位CPU下是占8个字节的。而指针类型的大小通常也是与CPU位数相关，一个指针所占用的内存在32位CPU下为4个字节，在64位CPU下也是8个字节。所以一个普通的iOS程序，从32位机器迁移到64位机器中后，虽然逻辑没有任何变化，但这种NSNumber、NSDate一类的对象所占用的内存会翻倍。而且为了存储和访问一个NSNumber对象，我们需要在堆上为其分配内存，另外还要维护它的引用计数，管理它的生命期。这些都给程序增加了额外的逻辑，造成运行效率上的损失。
![大神的图](https://upload-images.jianshu.io/upload_images/2427856-9f0d417b1616df42.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
为了改进上面提到的内存占用和效率问题，苹果提出了Tagged Pointer对象。由于NSNumber、NSDate一类的变量本身的值需要占用的内存大小常常不需要8个字节，拿整数来说，4个字节所能表示的有符号整数就可以达到21亿多(2^31)。所以我们可以将一个对象的指针拆成两部分，一部分直接保存数据，另一部分作为特殊标记，表示这是一个特别的指针，不指向任何一个地址。所以，实际上它不再是一个对象了，它只是披着对象皮的普通变量而已。所以，它的内存并不存储在堆中，也不需要malloc和free。假设你调用NSNumber的integerValue，它将从数据部分中提取数值并返回。这样，每访问一个对象，就省下了一次真正对象的内存分配，省下了一次间接取值的时间。同时引用计数可以是空指令，因为没有内存需要释放。对于常用的类，这将是一个巨大的性能提升。引入Tagged Pointer对象后，64位CPU下NSNumber内存图如下：
![大神的图](https://upload-images.jianshu.io/upload_images/2427856-93d3a6876206353c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
写下示例代码：

```
- (void)numberTest {
    NSNumber *number = [NSNumber numberWithInt:1];
    NSLog(@"%p",number);
}
```

>  StringTest[1309:126876] 0xb000000000000012

这里的指针地址中包含了对象特殊标记及指针指向的内容：地址0xb000000000000012最高四位的b就是NSNumber对象特殊标记，最低四位的2是用来标记number值的类型，2表示int类型(3:long,4:float,5:double)。而其余56位就是用来存储数值本身内容的，也就是说当数值超过56位存储上限的时候，那么NSNumber才会用真正的64位内存地址存储数值，然后用指针指向该内存地址。

```
- (void)numberTest {
    NSNumber *number1 = [NSNumber numberWithInt:1];
    NSNumber *number2 = [NSNumber numberWithLong:2];
    NSNumber *number3 = [NSNumber numberWithFloat:3];
    NSNumber *number4 = @(pow(2, 54));
    NSNumber *normalNumber = @(pow(2, 55));
    NSLog(@"%p\n%p\n%p\n%p\n%p",number1,number2,number3,number4,normalNumber);
}
```

>  0xb000000000000012
0xb000000000000023
0xb000000000000034
0xb400000000000005
0x604000038800

可以明显的看到，数值为2^55或更大时才在内存中分配一个NSNumber的对象来存储然后用指针指向该内存地址。可见，Tagged Pointer是可以与普通类共存的，即对一些值使用Tagged Pointer，另一些则使用一般的指针。

那么NSString对象同样也适用于Tagged Pointer。

```
- (void)stringTest {
    NSString *string1 = [NSString stringWithFormat:@"11"];
    NSString *string2 = [NSString stringWithFormat:@"a"];
    NSLog(@"%p\n%p",string1,string2);
}
```

> 0xa000000000031312
0xa000000000000611

和NSNumber一样，地址最高四位的a就是NSString对象特殊标记，而最低四位的是用来标记string的长度，其余56位就是用来存储字符串内容的（字符串内容转为为ASCII码存储）。我们能猜测当字符串所需内存小于56位时会使用Tagged Pointer，相反就会使用真正的NSString对象。实际情况就是如此吗？

```
- (void)stringTest {
    NSString *string1 = [NSString stringWithFormat:@"1234567"];
    NSString *string2 = [NSString stringWithFormat:@"12345678"];
    NSLog(@"%@---%p\n%@---%p",[string1 class],string1,[string2 class],string2);
}
```

> NSTaggedPointerString---0xa373635343332317
NSTaggedPointerString---0xa007a87dcaecc2a8

可以看到，string2内存64位，但还是使用了Tagged Pointer存储。只是编码的方式不一样了。具体的编码方式可以参考这篇[博客](http://www.cocoachina.com/ios/20150918/13449.html)，这里就简单列下不同字符串长度的编码方式：
> 1：如果长度介于0到7，直接用八位编码存储字符串。
2：如果长度是8或9，用六位编码存储字符串，使用编码表“eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX”。
3：如果长度是10或11，用五位编码存储字符串,使用编码表“eilotrm.apdnsIc ufkMShjTRxgC4013”

```
- (void)stringTest {
    NSString *string1 = [NSString stringWithFormat:@"123456789"];
    NSString *string2 = [NSString stringWithFormat:@"1234567890"];
    NSLog(@"%@---%p\n%@---%p",[string1 class],string1,[string2 class],string2);
}
```

> NSTaggedPointerString---0xa1ea1f72bb30ab19
__NSCFString---0x600000420060

当长度大于9时，使用真正的NSString对象存储。现在string2释放的问题就很明朗了：string2赋值为@"string string2"(长度：14)出了作用域后正常释放，string2赋值为@"string2"(长度：7)出了作用域不会释放。另外当字符串的内容有中文或者特殊字符（非 ASCII 字符）时，只能用NSString对象存储。字面量字符串不会使用Tagged Pointer。

#总结
我这里用了一道题来总结以上有关内存的内容：
在64位架构下，以下代码输出的结果是？

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self stringTest];
    NSLog(@"%@",_string1);
    NSLog(@"%@",_string2);
    NSLog(@"%@",_string3);
    NSLog(@"%@",_string4);
}

- (void)stringTest {
    NSString *string1 = @"1234567890";
    NSString *string2 = [NSString stringWithFormat:@"1"];
    NSString *string3 = [[NSString alloc] initWithFormat:@"2"];
    NSString *string4 = [[NSString alloc] initWithFormat:@"1234567890"];
    _string1 = string1;
    _string2 = string2;
    _string3 = string3;
    _string4 = string4;
}
```

相信大家很快就有了正确的答案。
写的比较杂，望多多指教。
