#第二章 对象 消息 运行期
##第6条：理解“属性”这一概念
属性是OC的一项特性，用于封装对象中的数据（通常数据保存为各种实例变量）。实例变量一般通过存取方法getter和setter方法读取和写入，使用属性，编译器自动编写了相关的存取方法（自动合成），我们也可以自己编写存取方法，但一定要满足命名规范。使用属性，编译器还自动向类中添加了适当类型的实例变量，并在属性名前加_作为实例变量的名称。同时属性引入了“点语法”（点语法其实就是调用了getter和setter方法），我们可以容易的访问存放于属性中的数据。
属性有关的关键字：@synthesize @dynamic
@synthesize可以指定实例变量的名称，@dynamic编译器不会创建实现属性的实例变量，也不会合成存取方法。
属性的特质会影响到存取方法，属性拥有四种特质：
1. 原子性：atomic是“原子的”，合成的方法中会加同步锁。nonatomic是“非原子的”，合成方法不会加同步锁。一般情况下使用nonatomic，因为加同步锁意义不大，因为并不能确保线程安全，而且使用同步锁开销较大，会带来性能问题。
2. 读写权限：readwrite,属性拥有setter和getter方法；readonly，属性仅拥有获取方法，外部不能更改数据。
3. 内存管理语义：assign,setter方法只会执行针对“纯量类型”的简单赋值操作；strong,属性定义了一种拥有关系，setter方法会保留新值，释放旧值，再将新值设置上去；weak,属性定义了一种非拥有关系，setter方法不会保留新值，也不释放旧值，当属性指向的对象销毁时，属性值会自动清空；unsafe_unretained,和weak类似，区别是当属性指向的对象销毁时，属性值不会自动清空（不安全）；copy,和strong类似，然而setter方法并不保留新值，而是将其拷贝。当类型有可变和不可变类型时，不可变的属性一般需要使用copy特质，因为设置的新值可能指向可变的类型，如果不用copy，属性值可能在不知情的情况下遭人修改。
4. 方法名：getter=<name> 指定获取方法的方法名；setter=<name>指定设置方法的方法名。
##第7条：在对象内部尽量直接访问实例变量
直接访问实例变量和通过属性访问实例变量区别：
1. 直接访问实例变量速度相对较快，因为不需要经过“方法派发”。编译器所生成的代码会直接访问保存对象实例变量的那块内存。
2. 直接访问实例变量，不会调用getter，setter方法，这就绕过了属性所定义的内存管理语义。（修饰属性的strong,copy等没用了）
3. 直接访问实例变量，不会触发KVO。
4. 通过属性访问实例变量，可以通过getter，setter方法监控属性的调用者和其访问时机，方便调试。

综合以上，折中的选择是：
1. 读取数据时，应该直接通过实例变量读取；写入数据时，应该通过属性写入（setter方法）。
2. init方法及dealloc方法中，应该直接通过实例变量来读写数据。
3. 使用懒加载时，必须通过属性读写数据，而懒加载方法内必须直接通过实例变量读取数据（避免死循环）。
4. 使用KVO时，需通过属性读取数据。
##第8条：理解“对象等同性”这一概念
关于“等同性”，==操作符比较的是两个指针本身，而不是其所指的对象。比如我们要判断俩个字符串是不是一样的，就不能使用==操作符，而必须使用内建的isEqualToString:方法。NSObject协议提供了两个判断等同性的关键方法：
```
- (BOOL)isEqual: (id) object;
- (NSUInteger)hash;
```
我们可以实现协议的这两个方法，实现我们定义的“等同性”，类似于NSString类的isEqualToString:
1. isEqual: 方法实现判断相等的条件。
2. 如果isEqual: 方法判定两个对象相等（返回YES）,那么hash方法也必须返回同一个值。
3. 编写hash方法时，应该使用计算速度快而且哈希码碰撞几率低的算法。最常用的算法是把类的每个判等的属性的哈希值进行异或运算，这样做既能保持较高的效率，又能使生成的哈希码至少位于一定范围内，而不会频繁的重复。
##第9条：以类族模式隐藏实现细节
类族（类簇）:基类提供创建各子类的方法，并提供接口，子类通过接口实现相应的方法。用户无需自己创建子类实例，只需要调用基类方法创建即可。这样将子类的实现细节隐藏在抽象基类后面。工厂模式是创建类族的方法之一。系统框架中普遍使用类族，例如NSArray(大部分集合类),NSArray的alloc方法获取实例时，该方法首先会分配一个属于某类的实例充当“占位数组”。该数组稍后会转为另一个类的实例，而这个类就是NSArray的实体子类。
又如UIButton:
```
+ (instancetype)buttonWithType:(UIButtonType)buttonType;
```
UIButton通过这个类方法创建button，button的类型取决于传入的按钮buttonType，他会返回对应类型的UIButton的子类对象，它们的基类都是UIButton。而具体的每个子类的实现细节，即各种类型button的创建过程不需要我们考虑，我们只需调用基类的方法即可。
类族的一个注意点：类似NSArray alloc创建的对象，其实返回的是对应的子类而非NSArray基类，所以判断创建的对象是否是NSArray类时会有问题：
```
- (BOOL)isKindOfClass:(Class)aClass; // YES
- (BOOL)isMemberOfClass:(Class)aClass; // NO
```
##第10条：在既有类中使用关联对象存放自定义数据
1. 可以通过objc_setAssociatedObject方法给对象关联其他对象，objc_getAssociatedObject获取关联的对象。
```
objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key,
                         id _Nullable value, objc_AssociationPolicy policy)
```
参数object为关联的对象；key是区分关联对象的“键”，key一般需要声明为static全局变量；policy为存储策略，是用来维护相应的“内存管理语义”，和属性的内存管理语义是等效的。
2. 通过关联对象，我们可以在分类中实现添加属性的功能。
3. 只有在其他做法不可行时才应选用关联对象，因为这种做法通常会引入难以查找的bug
##第11条：理解objc_msgSend的作用
OC是一门消息型语言，在对象上调用方法，就是传递消息的过程。消息有“名称name”或“选择器selector”,可以接受参数而且可能有返回值：消息由接收者,选择子及参数构成。如果向某对象传递消息，那就会使用“动态绑定”机制来决定需要调用的方法，究竟该调用哪个方法完全于运行期决定，甚至在程序运行时改变，这使得OC能成为一门真正的动态语言。发送消息时，编译器会将消息转换为标准的C语言函数调用，这个函数即消息传递机制中的核心函数objc_msgSend，其原型如下：
```
void objc_msgSend(id self, SEL cmd, ...)
```
这是个可变参数的函数，能接受两个及以上的参数。第一个参数为接受者，第二个参数cmd为选择器，后续参数为消息本来的那些参数，顺序也一致。
objc_msgSend函数会依据接受者与选择器来调用适当的方法：首先需要在接收者所属的类中搜寻“方法列表”（选择器的名称则是查表所用的key），如果能找到与选择器名称相符的方法，就跳至对应的实现代码。若是找不到，那就沿着继承体系向上查找，找到了合适的方法后再跳转。如果最终找不到，就会执行消息转发操作。
##第12条：理解消息转发机制
当对象接收到无法解读的消息后（对象无法响应某个选择器），则进入消息转发流程。
消息转发分为以下阶段：
1. 动态方法解析：这时可以通过 + (BOOL)resolveInstanceMethod:(SEL)sel方法动态添加一个方法。
2. 备援接收者：当前面一个阶段仍没处理时，会有第二次机会处理未知的选择器，这一步可以通过- (id)forwardingTargetForSelector:(SEL)aSelector返回备援对象将消息转给其他接收者来处理。
3. 完整的消息转发：当以上两步都未处理时，那么唯一能做的就是启用完整的消息转发机制。首先通过- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector方法返回函数签名由后续方法- (void)forwardInvocation:(NSInvocation *)anInvocation执行：NSInvocation对象anInvocation里封装了未处理的消息有关的全部细节（选择器，target及参数）。通过这个方法可以某种方式改变消息内容，如添加参数，更换选择器，替换target（和备援接收者等效）等。
4. 当前面几步都未处理消息时，会调用- (void)doesNotRecognizeSelector:(SEL)aSelector以抛出异常，表明选择器最终未能得到处理。
![盗得图](https://upload-images.jianshu.io/upload_images/2427856-ccdda21a5a7e4e14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
接收者在每一步中都有机会处理消息，步骤越后，处理消息的代价就越大。最好在前面的步骤完成处理。
##第13条：黑魔法：“方法调配”（method swizzling）
与给定的选择器名称相对应的方法实现也可以在运行期改变，通过这个特性，我们既不需要源代码也不需要通过继承子类来覆写就能改变这个类的本身功能。这种方案就是“方法调配”（method swizzling）。
类的方法列表会把选择器的名称映射到相关的方法实现上，使得“动态消息派发系统”能够据此找到应该调用的方法，这些方法均以函数指针的形式来表示，这种指针叫做IMP，
```
id (* IMP) (id, SEL, ...)
```
OC运行期系统提供了几个方法能够操作这张表：
```
// 新增方法
class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp,  const char * _Nullable types) 
// 交换方法
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2) 
```
![timg.jpg](https://upload-images.jianshu.io/upload_images/2427856-b01b1aafd45cedc5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
但是这种方法不宜滥用，若是滥用反而令代码变得不易读懂且难于维护。
##第14条：理解“类对象”的用意
每个OC对象实例都是指向某块内存数据的指针，所以在声明变量时，类型后面要跟一个“*”。对于通用对象类型id，由于它本身已经是指针了，不用带“*”。
描述OC对象使用的数据结构如下：
```
typedef struct objc_object *id;
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```
由此可见，每个对象结构体都有一个指向Class对象的指针isa（通常称为“is a”指针）,其定义了对象所属的类，表明其类型，而Class对象则构成了类的继承体系。Class结构如下：
```
typedef struct objc_class *Class;
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```
结构体存放类的元数据：类的实例变量（ivars），类的实例实现方法（methodLists）等信息。同样，它也有isa指针，这说明Class本身也是OC对象，这个isa指针指向的是类对象所属的类型，叫做“元类”。类方法就定义在“元类”中。super_class定义了本类的超类。
![类继承体系](https://upload-images.jianshu.io/upload_images/2427856-7f1434bfabc0f33f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在类继承体系中查询类型信息：
1. isMenberOfClass 能够判断对象是否为某个特定类的实例；
2. isKindOfClass 能够判断出对象是否为某类或其派生类的实例。
（第9条也有提及）
