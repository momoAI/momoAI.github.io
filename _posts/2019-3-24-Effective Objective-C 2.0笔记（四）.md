##第五章 内存管理
###第29条：理解引用计数
- OC使用引用计数管理内存，引用计数机制通过递增递减的计数器来管理内存。对象创建后，保留计数为1。若保留计数为正，则对象继续存活，保留计数降为0时，对象被销毁
- 对象有三个方法用于操作计数器：`retain`:递增保留计数，`release`:递减保留计数，`autorelease`:待自动释放池销毁时，再递减保留计数；
- MRC下对象调用release，保留计数如果降为0，对象所占内存会被回收；再访问对象**可能**使程序崩溃;因为对象所占内存dealloc后，只是放回“可用内存池”中；若果对象内存未被覆写，那么该对象仍然有效，程序不会崩溃；反之才会造成崩溃；为了防止访问野指针造成的程序崩溃，在对象release后应手动将指针置nil
```
[object release];
object = nil;
```
- 属性存取方法中的内存管理：若属性为“strong”类型，setter方法的处理方式为：保留新值再释放旧值，然后更新实例变量令其指向新值；
```
- (void)setFoo:(Foo *)foo {
  [foo retain];
  [_foo release];
  _foo = foo;
}
```
这个顺序极其重要，假如还未保留新值就释放旧值，两个对象又指向了同一个对象，那么先执行的release操作可能导致系统将此对象永久回收，后续的retain操作则无法令这个被彻底回收的对象复生；
- autorelease对象的释放时机：如果创建了自己的自动释放池，对象会在自动释放池销毁时释放（即出了@autoreleasepool {    }作用域后释放）；否则就是等到当前线程的下一次事件循环（Runloop）时释放，因为一次事件循环会开启新的自动释放池；由此，autorelease能延长对象生命周期；
###第30条：以ARC简化引用计数
- ARC会自动执行retain, release,autorelease等内存管理操作，在ARC下调用remain,release,autorelease,dealloc,ratinCount等方法是非法的，编译会报错；
- ARC在调用内存管理方法时，是直接调用底层C语言，这样性能更好，因为保留，释放操作比较频繁，直接调用底层函数能节省很多CPU周期；
- ARC确定的硬性规定：方法名以`alloc`,`new`,`copy`,`mutableCopy`开头的方法创建对象，返回的对象归调用者所有（负责释放对象）；若方法名不以这四个词语开头，则表示所返回的对象不归调用者所有，返回的对象会自动释放；实际上就是调用了autorelease方法；因为ARC的这个硬性规定，所以声明属性名称时不能以new等开头，因为这样属性setter方法的内存语意会分不清；
- ARC也会执行手工操作无法完成的优化：在编译期，ARC会把能够相互抵消的retain，release，autorelease操作约简，如果发现同一个对象上执行了多次保留与释放操作，ARC有时可以成对移除这两个操作；
- 应用程序中，可用下列修辞符来改变局部变量与实例变量的语义：
`__strong`:默认语义，保留值；
`__unsafe_unretained`:不保留值，变量不会自动清空，可能不安全，会发生野指针问题；
`__weak`:不保留值，但变量会自动清空，是安全的；
`__autorelease`:自动释放；
- ARC只负责管理OC对象的内存，对于OC对象，ARC会自动生成回收对象所执行的代码，但是对于非OC对象，如CoreFoundation中的对象或由malloc()分配在堆中的内存，那么仍需手动清理。
```
- (void)dealloc {
  CFRelease(_coreFoundationObject);
  free(_heapAllocaatedMemoryBlob);
}
```
###第31条：在dealloc方法中只释放引用并解除监听
在dealloc方法中，应该做的是：释放指向其他对象的引用；移除KVO或NSNotificationCenter通知；
###第32条：编写“异常安全代码”时留意内存管理问题
异常一般只应在严重错误后才抛出（第21条）；
在@try块中，如果先保留了某个对象，然后在释放它之前又抛出异常，除非@catch块能处理该问题，否则就发生内存泄露了；
- MRC模式下解决方法：在@finally块中调用release释放对象，因为@finally块，无论是否抛出异常，代码都会运行；
```
    Object *ojbect;
    @try {
        object = [[Object alloc] init];
        exception
        ....
    }
    @catch(NSExpression *expression) {
        ...
    }
    @finally {
        [object release];
    }
```
- ARC模式，默认情况下ARC不会自动处理这种情况，又不能调用release。对于异常处理更加棘手；解决方法是：通过设置`-fobjec-arc-exceptions`编译标志来开启ARC生成安全处理异常代码的模式；这可以使ARC安全处理异常；
###第33条：以弱引用避免保留环（循环引用）
- 循环引用会导致内存泄露，避免循环引用的最佳方式就是弱引用；这种引用表示“非拥有关系”；
- “非拥有关系”的修辞符有`weak`,`assign`,`unsafe_unretained`;assign一般用于基本类型（int ,float,struct），unsafe_unretained一般用于对象类型，是不安全的；weak也用于对象类型，但是安全的；assign也可以用于对象类型，但也是不安全的；weak不能用于基本类型;
- weak之所以安全，是因为变量指向的实例被释放回收时，weak属性会指向nil，而unsafe_unretained属性仍然指向已被回收的对象，这时在访问属性时会发生错误；
- weak属性自动置为nil的原理：weak属性对象会被写入一张哈希表中，以weak属性指向的对象地址为key，weak指针为value；当指向的对象销毁时，会根据对象地址去表中查找weak指针并置为nil;
###第34条：以自动释放池降低内存峰值
非alloc,new,copy,mutableCopy词开头方法创建的对象，都是autorelease对象，如果没有手动创建自动释放池，那么autorelease对象要等到下个Runloop后才释放；如果在下个Runloop之前，autorelease对象已经很多（比如for循环创建对象），那么内存峰值将会很高；那么，可以在适当的时机，手动创建autorelease pool使得对象及时释放以降低内存：
```
      for (int i = 0; i < 100000; i ++) {
            @autoreleasepool {
                Object *obj = [Object ojbect];
            }
        }
```

###第35条：用“僵尸对象”调试内存管理问题
- 僵尸对象：内存已被回收的对象；指向僵尸对象的指针就是野指针
- 向僵尸对象发送消息是不安全的，有可能造成程序崩溃；崩溃与否，取决于对象所占内存有没有为其他内容所覆写；如果内存未被覆写，那访问仍然有效；如果被覆写了，消息会发送给新对象，新对象可能刚好能应答那访问也有效；（29条有类似描述）
- 可以使用Xcode开启僵尸对象选项，避免访问僵尸对象发生的不安全行为；

![zombie](https://upload-images.jianshu.io/upload_images/2427856-7c3da7e5cff04686.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 僵尸对象选项原理：勾选Zombie Object选项后，运行时系统会把已经回收的实例转化为特殊的“僵尸对象”，而不会真正回收它们。没有回收，也就不可能被覆写。僵尸对象收到消息后，会抛出异常，且准确说明发送的消息及描述回收前的那个对象信息；
- Zombie Object运行时系统处理过程：系统在回收对象时，如果发现环境变量设置了Zombie Object选项，那么会把该对象转化为有“僵尸”标识的对象；具体做法是：使用黑魔法swizzle method替换dealloc方法，通过这个dealloc方法动态的创建一个以`_NSZombie_`前缀+原类名为类名的新类，然后将这个新建的类设置为原实例的class，对象就变成了“僵尸”标识的类；`_NSZombie_`前缀的新类，并未实现任何方法，所以发给它的消息就要经过“消息转发”，这个过程中将收到的消息及原来所属的类打印出来，抛出异常，程序终止。（整个过程，其实和KVO实现过程有点类似）

###第36条：不要使用retainCount
