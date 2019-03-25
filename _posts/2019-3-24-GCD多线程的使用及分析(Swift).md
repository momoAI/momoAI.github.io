![timg.jpg](https://upload-images.jianshu.io/upload_images/2427856-b21820e8c7999d36.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


GCD全称Grand Central Dispatch，从名称可以看出GCD就是起到中央调度的作用。这个调度作用就是对线程做的处理：GCD 会自动管理线程的生命周期（创建线程、调度任务、销毁线程），程序员只需要告诉 GCD 想要执行什么任务，不需要编写任何线程管理代码。
在使用GCD多线程编程前，我们需要理解两个概念：队列，任务。

# 队列与任务

```
        let queue = DispatchQueue.init(label: "queue_label")
        queue.async {
            print("task")
        }
```

queue就是我们创建的GCD队列，async后面的闭包就是我们添加到队列里的任务。队列负责调度线程，而任务的执行会根据是同步sync还是异步async来开启线程。
GCD其实已经弱化了线程的概念，转而使用了队列这个东西。GCD队列其实就是对线程的封装，所有对线程的操作已经交由队列处理，而且还结合性能的开销情况帮我们做了一系列优化。我们不用再像使用NSThead那样管理线程，GCD可以做到更高效性能更佳。我们只要在闭包写好要多线程处理的任务，然后扔给队列就行了。至于怎么开启线程，开多少线程都交由GCD去完成了。
任务分为同步执行和异步执行。
# 同步执行

```
        queue.sync {
            // task
        }
```

同步执行不会开启线程，任务执行需要等待前面任务完成。
# 异步执行

```
        queue.async {
            // task
        }
```

异步执行会开启线程，任务执行无需等待并发完成。对于线程，并不是有多少个任务就开启多少个线程。开启一个线程是会消耗CPU的资源的，GCD并不会帮我们开启太多线程，它会合理的控制线程数。对于多个线程，GCD会创建类似UITableView缓存池的线程池，达到重用的目的也不需要开启那么多线程。这个接下来会具体分析。
我们再来看看GCD的队列
![queue](https://upload-images.jianshu.io/upload_images/2427856-c5b2e270bfbcac82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
GCD的队列分为串行队列和并行队列，系统提供了两个队列:main,global。main为主队列属于串行队列，在主线程执行。global为全局并行队列。顾名思义串行队列就是按顺序一个一个执行任务的，而并行队列是并发处理任务的。我们都知道队列是FIFO先进先出的，那么并发队列为何能并发处理呢？我们来看看串行和并行队列的本质
#串行队列
前面说了，队列负责调度线程。而串行队列只能调度一个线程，但并行队列是可以调度多个线程的。这就是它们本质的区别。
![线程调度](https://upload-images.jianshu.io/upload_images/2427856-e4d356538fc0ddb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
串行队列里的任务是需要等待队列里前面一个任务结束了才老老实实开始的，就好比我们去车站排队买票，前面那个同志付钱买好了才轮得到你。用图来描述一下：
![串行队列](https://upload-images.jianshu.io/upload_images/2427856-58a1a47c23a6e636.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
写个函数验证下：

```
  func sQueueASync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label")
        queue.async {
            print("task1!----\(Thread.current)")
        }
        queue.async {
            sleep(3)
            print("task2!----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task3!-----\(Thread.current)")
        }
        queue.async {
            print("task4!-----\(Thread.current)")
        }
        
        print("End")
    }
```

输出结果如下
> Hello, World!
End
task1!----<NSThread: 0x15662220>{number = 2, name = (null)}
task2!----<NSThread: 0x15662220>{number = 2, name = (null)}
task3!-----<NSThread: 0x15662220>{number = 2, name = (null)}
task4!-----<NSThread: 0x15662220>{number = 2, name = (null)}

 func sQueueASync() -> Void {}函数里面的操作，其实就是添加在主队列里的一个任务，主队列使用的就是主线程。queue队列使用async异步执行会开启子线程，不会阻塞当前的主线程，所以主队列中“End”不会受task1,2,3,4影响。在输出“Hello, World!”后马上执行了打印“End”的任务。而在我们手动创建的queue队列中，虽然使用async异步开启多个线程执行，但结果4个task都只使用了同一个线程。由于串行队列中任务必须等前面任务完成后才开始的，所以不管task2,task3睡多久，都是按照1，2，3，4的顺序执行的。异步执行可以开启多个线程，但因为串行队列只能调度一个线程，也没必要开启多个线程了。所以在某个时间点，串行队列的任务只会开启一个线程(主队列又不一样了，因为主队列已经有了主线程不会再开启线程)。把手动创建的串行队列改成主队列看看：

```
  func sQueueASync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.main
        queue.async {
            print("task1!----\(Thread.current)")
        }
        queue.async {
            sleep(3)
            print("task2!----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task3!-----\(Thread.current)")
        }
        queue.async {
            print("task4!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!
End
task1!----<NSThread: 0x17e450c0>{number = 1, name = main}
task2!----<NSThread: 0x17e450c0>{number = 1, name = main}
task3!-----<NSThread: 0x17e450c0>{number = 1, name = main}
task4!-----<NSThread: 0x17e450c0>{number = 1, name = main}

可以看到，执行的顺序和上面是一样的，因为都是串行队列，只是都是在主线程执行了。也就是说，在主队列中也只能调度一个线程，且这个线程为主线程。
大家肯定注意到了，前面着重强调了"在某个时间点"这个条件。这个条件是什么意思呢？这和Runloop有关系。GCD会有一个线程的缓存池，类似UITableView的缓存池起到重用线程的作用。在Runloop一个循环内，线程池不会销毁会一直存在，缓存池中线程可以供下个任务使用。在下个循环开始，线程池销毁，池里没有可用线程就需要重新再开启一个线程了。也就是说在"另一个时间点"需要再开启线程。可以模拟一下这种情况:

```
func sQueueASync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label")
        queue.async {
            print("task1!----\(Thread.current)")
        }
        queue.async {
            sleep(3)
            print("task2!----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task3!-----\(Thread.current)")
        }
        queue.asyncAfter(deadline: DispatchTime.now() + 30) {
            print("task4!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!
End
task1!----<NSThread: 0x155a1e10>{number = 2, name = (null)}
task2!----<NSThread: 0x155a1e10>{number = 2, name = (null)}
task3!-----<NSThread: 0x155a1e10>{number = 2, name = (null)}
task4!-----<NSThread: 0x155ae030>{number = 3, name = (null)}

这里把task4稍微改动了下，加了30s延迟，结果是task4重新开启了新的线程。30s后其实是处于另一个循环了，之前的task1,2,3的线程已销毁。

前面分析的都是串行队列异步执行，接下来再看看串行队列同步执行的情况。还是上面的代码，把异步async改成同步sync。

```
  func sQueueSync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label")
        queue.sync {
            print("task1!----\(Thread.current)")
        }
        queue.sync {
            sleep(3)
            print("task2!----\(Thread.current)")
        }
        queue.sync {
            sleep(2)
            print("task3!-----\(Thread.current)")
        }
        queue.sync {
            print("task4!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!
task1!----<NSThread: 0x14570ac0>{number = 1, name = main}
task2!----<NSThread: 0x14570ac0>{number = 1, name = main}
task3!-----<NSThread: 0x14570ac0>{number = 1, name = main}
task4!-----<NSThread: 0x14570ac0>{number = 1, name = main}
End

queue队列里的4个task和主队列异步执行的情况基本一致：任务按顺序一个一个执行且没有开启新的线程。唯一不同的就是主队列"End"任务的顺序。这里的"End"任务一定是最后执行的，因为串行队列同步执行没有开启线程，同步执行的任务会阻塞主线程，"End"任务需要等待4个task任务执行完才执行。而且由于queue是我们手动创建的串行队列，和主队列不会造成互相等待，不会造成死锁。
同样的，把上面代码中手动创建的串行队列换成main队列，看看是什么结果

```
  func sQueueSync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.main
        queue.sync {
            print("task1!----\(Thread.current)")
        }
        queue.sync {
            sleep(3)
            print("task2!----\(Thread.current)")
        }
        queue.sync {
            sleep(2)
            print("task3!-----\(Thread.current)")
        }
        queue.sync {
            print("task4!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!

输出“Hello, World!”后程序卡死，这段代码就发生了死锁。发生死锁的原因是什么？画个图分析下：
![死锁](https://upload-images.jianshu.io/upload_images/2427856-e4ffd0ace34129f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4个task添加至主队列在主线程执行，而sQueueSync()本身就是主队列里的一个任务，所以主队列现在可以看做有5个任务。而且这5个任务是是按顺序执行的。而sQueueSync()里main队列任务可以分成3块：1,打印"Hello,World!"，2.把task1,2,3,4添加至主队列执行，3.打印"End"。很明显这是一个嵌套的任务。前面分析了，串行队列是必须等到前面任务完全执行完后才执行的，main队列中Hello, World!执行完后，接着准备执行4个task，等待执行完4个task后才会执行"End"任务。而这时4个task的任务必须等到队列中sQueueSync()任务完成后才开始执行，这就造成了相互等待，死锁发生了。
到这里，基本把串行队列的情况都列举了下。再回顾下，串行队列有以下4种情况。
- 串行队列异步执行
- 主队列异步执行
- 串行队列同步执行
- 主队列同步执行
歇了一口气，接着讲并行队列
# 并行队列
并行队列可以调度多个线程，它可以并发的执行多个任务而无需等待前一个任务完成。
![调度线程](https://upload-images.jianshu.io/upload_images/2427856-06b9bb8a6af5fee6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

咦？队列不是先进先出按顺序一个一个来的吗？没错，并发队列也还是先进先出按顺序的，这里说的并发只是说前面一个任务刚执行还没结束下一个任务马上开始，这中间的速度很快。看起来就像是并发一起执行的，这里只是相对串行队列需要等待前面一个任务执行完后才执行而言的。同样我们还是拿买车票做比喻：我们还是去车站排队买票，不同的是这时排队的窗口不止一个售票员。那么我们前面的那个同志小明先我们一步去了售票员A那里买票了，这时售票员B是空闲的，我们就没必要傻傻的非要等到小明买完了结清账了再去A那买票，我们就直奔B那里了。而小明可能需要买很多张票，他可能帮全家人一起买，而我们潇潇洒洒就买了一张票。等我们买好了出来了，发现小明那里还在刷身份证。这时你不能说我插了小明的队吧，我们还是老老实实排队来的。所以说并行队列也还是按顺序执行的，只是它不需要等待，而且哪个先执行完也是不确定的。还是作个图清楚些：
![并行队列](https://upload-images.jianshu.io/upload_images/2427856-0fd582f892574b55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对比之前串行队列的图示，区别还是比较明显的。
和串行队列一样，并行队列也分同步执行、异步执行。

```
    func cSync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label", attributes: .concurrent)
        queue.sync {
            print("task11!----\(Thread.current)")
        }
        queue.sync {
            sleep(3)
            print("task22!----\(Thread.current)")
        }
        queue.sync {
            print("task33!-----\(Thread.current)")
        }
        queue.sync {
            print("task44!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!
task11!----<NSThread: 0x14e3a7c0>{number = 1, name = main}
task22!----<NSThread: 0x14e3a7c0>{number = 1, name = main}
task33!-----<NSThread: 0x14e3a7c0>{number = 1, name = main}
task44!-----<NSThread: 0x14e3a7c0>{number = 1, name = main}
End

这里创建了.concurrent的并行队列，而且是同步执行。从输出结果可以看出任务都是按顺序执行的，但前面我们分析了并行队列是不需要等待前面任务完成后再执行的，按理说task2需要3s其他task能立即执行完，但task3和task4还是在task2后执行。其实任务是顺序执行还是并发执行是要结合队列类型及任务执行方式综合分析的，如同之前的串行队列异步执行：虽然异步会开启线程任务应该能并发执行，但由于是串行队列只会调度一个线程任务也需要等待，结果还是按顺序执行了。这里的并行队列同步执行道理是一样的：虽然并行队列能调度多个线程，但由于是同步的方式也就不会开启线程它会阻塞线程。并行队列没有可以调度的线程，也就是仍需等待前面任务完成才能开始。还是举买票这个栗子，我们排队的窗口本来是可以有多个售票员的，但这时车站的人员不够也就只安排了一个售票员。那这时我们还是得等前面的小明买好票离开才开始。现在不管小明花多长时间，他就算一直和售票员跑火车，我们也没办法还是得等他。现在看着小明是不是很不顺眼。所以说串行队列并行队列和同步异步方式都是相互关联的，不能说并发队列或异步执行就一定会并发处理。既然如此，现在把sync改成async试试：

```
func cAsync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label", attributes: .concurrent)
        queue.async {
            print("task11!----\(Thread.current)")
        }
        queue.async {
            sleep(3)
            print("task22!----\(Thread.current)")
        }
        queue.async {
            print("task33!-----\(Thread.current)")
        }
        queue.async {
            print("task44!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!
End
task11!----<NSThread: 0x1758fe10>{number = 2, name = (null)}
task33!-----<NSThread: 0x17673220>{number = 3, name = (null)}
task44!-----<NSThread: 0x1758fe10>{number = 2, name = (null)}
task22!----<NSThread: 0x17580d70>{number = 4, name = (null)}

结果在意料之中，主线程没有被阻塞，Hello, World!  End执行不受queue里任务影响。queue队列开启了3个线程，任务是并发执行的。这里也验证了之前说的另一个结论：并不是有多少任务就开启多少线程的，task1和task4都是线程2。为了更明显看出这种结果，现在在原先代码了多增加几个任务。

```
func cAsync() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label", attributes: .concurrent)
        queue.async {
            print("task11!----\(Thread.current)")
        }
        queue.async {
            sleep(3)
            print("task22!----\(Thread.current)")
        }
        queue.async {
            print("task33!-----\(Thread.current)")
        }
        queue.async {
            print("task44!-----\(Thread.current)")
        }
        queue.async {
            print("task55!-----\(Thread.current)")
        }
        queue.async {
            print("task66!-----\(Thread.current)")
        }
        queue.async {
            print("task77!-----\(Thread.current)")
        }
        
        print("End")
    }
```

> Hello, World!
End
task11!----<NSThread: 0x1457fbe0>{number = 2, name = (null)}
task33!-----<NSThread: 0x1457fbe0>{number = 2, name = (null)}
task44!-----<NSThread: 0x1457fbe0>{number = 2, name = (null)}
task55!-----<NSThread: 0x1457fbe0>{number = 2, name = (null)}
task66!-----<NSThread: 0x145800b0>{number = 3, name = (null)}
task77!-----<NSThread: 0x1457fbe0>{number = 2, name = (null)}
task22!----<NSThread: 0x1457c330>{number = 4, name = (null)}

现在是7个任务，但还是3个线程。但这段代码最好在真机上运行调试，你可以在模拟器上试下，会发现我被啪啪啪打脸：模拟器上运行有多少任务就开启了多少线程。应该是因为模拟器的CPU和内存和电脑一样的缘故，具体原因还没深入研究。
至此，GCD串行队列，并行队列，同步执行，异步执行的组合情况都分析完毕了。现再总结下：
- 串行队列异步执行
- 主队列异步执行
- 串行队列同步执行
- 主队列同步执行
- 并行队列异步执行
- 并行队列同步执行
1. 串行队列同步执行没有任何实际意义
2. 并行队列同步执行会阻塞当前线程，所有任务都按顺序执行，没有任何实际意义
3. 主队列同步执行会死锁
4. 串行队列异步执行会开启线程，不会阻塞主线程，队列里的任务顺序执行
5. 并行队列异步执行会开启线程，不会阻塞主线程，队列里的任务并发执行

列个表格总结下
![总结](https://upload-images.jianshu.io/upload_images/2427856-8b7d605c95be00c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，串行队列异步执行和并行队列异步执行才是最实用的，这也是我们平时项目里用的最多的。具体用哪种，就要看你是想并发执行还是顺序执行了。
有了这个基础后，我们再来分析嵌套任务的情况：
# 嵌套任务
之前的主队列同步/异步执行就属于嵌套任务的情况，只是嵌套的是主队列。我们看下自己创建的队列嵌套任务的情况：

```
func nestQueue1() -> Void {
        print("Hello, World!")
        let squeue = DispatchQueue.init(label: "queue_label")
        squeue.sync {
            print("task11!----\(Thread.current)")
            squeue.sync {
                print("task22!----\(Thread.current)")
            }
            print("task33!-----\(Thread.current)")
        }
        print("End")
    }
```

这里还是串行队列同步执行，不过添加至squeue队列里的任务（即squeue.sync{}闭包）本身又追加了一个同步任务到squeue。我们先不管主队列的print("Hello, World!")和print("End")，单看squeue.sync {}这一块，是不是似曾相识？没错这就和上面的主队列同步执行一样，只是队列换成了我们自己创建的串行队列。除去print("Hello, World!")和print("End")，其实它就等价于下面代码

```
func nestQueue1() -> Void {
        print("task11!----\(Thread.current)")
        let squeue = DispatchQueue.main
        squeue.sync {
             print("task22!----\(Thread.current)")
        }
        print("task33!-----\(Thread.current)")
    }
```

毫无疑问，它也会发生死锁：
> Hello, World!
task11!----<NSThread: 0x16546a70>{number = 1, name = main}

接下来，再把刚才嵌套的同步任务换成异步试试：

```
func nestQueue2() -> Void {
        print("Hello, World!");
        let squeue = DispatchQueue.init(label: "queue_label")
        squeue.sync {
            print("task11!----\(Thread.current)")
            squeue.async {
                print("task22!----\(Thread.current)")
            }
            sleep(3)
            print("task33!-----\(Thread.current)")
        }
        print("End")
    }
```

> Hello, World!
task11!----<NSThread: 0x15d1d120>{number = 1, name = main}
task33!-----<NSThread: 0x15d1d120>{number = 1, name = main}
End
task22!----<NSThread: 0x15e4d1a0>{number = 2, name = (null)}

由于squeue.sync {}里嵌套的是squeue.async{}不会造成相互等待，task2也在另一个线程执行，不会造成死锁。最外层squeue.async{}是同步执行会阻塞主线程，所以主队列的print("End")任务需要等待squeue里任务执行完，也就是说“End”一定是在task3之后。嵌套的task2是异步执行的但是添加至同一个串行队列squeue中，根据前面分析的串行队列异步执行的情况，task2不会阻塞当前任务执行但并也不能并发执行，所以task2还是需要等待前面的任务执行完后才开始。也就是说task2也肯定会是在task3后。还是看图会比较清楚些：
![嵌套情况](https://upload-images.jianshu.io/upload_images/2427856-7602b8a5fca1bd60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

把刚才的代码内层和外层的同步和异步对调下：

```
func nestQueue3() -> Void {
        print("Hello, World!");
        let squeue = DispatchQueue.init(label: "queue_label")
        squeue.async {
            print("task11!----\(Thread.current)")
            squeue.sync {
                print("task22!----\(Thread.current)")
            }
            print("task33!-----\(Thread.current)")
        }
        print("End")
    }
```

> Hello, World!
End
task11!----<NSThread: 0x15d612a0>{number = 2, name = (null)}

很不幸的是，发生了死锁。
squeue最外层的任务为异步执行，主队列不会受到影响。死锁就是发生在squeue.async {}里面 。squeue嵌套的任务squeue.sync{}是同步执行,且是串行队列这就造成了squeue.async {}与squeue.sync {}相互等待。
以上都是串行队列嵌套的情况，串行队列的任务需要等待，所以嵌套的任务是同步执行的话都会发生死锁。当然，还有并行队列嵌套的情况，这里就不具体分析了。直接来看另外几个实用的功能吧;
#DisPatchGroup
DisPatchGroup是GCD中一个组的概念，可以把相关的任务归并到一个组内来执行，通过监听组内所有任务的执行情况来做相应处理。

```
func group() -> Void {
        print("Hello, World!");
        let group = DispatchGroup()
        let queue = DispatchQueue.init(label: "queue_label", attributes: .concurrent)
        queue.async(group:group) { // 将任务添加至group组内
            sleep(2)
            print("task11!----\(Thread.current)")
        }
        queue.async(group:group) {
            sleep(2)
            print("task22!----\(Thread.current)")
        }
        group.notify(queue: queue) { // 监听任务组内任务的执行情况
            print("task33!----\(Thread.current)")
        }
        print("End")
    }
```

> Hello, World!
End
task22!----<NSThread: 0x145622d0>{number = 2, name = (null)}
task11!----<NSThread: 0x14649c40>{number = 3, name = (null)}
task33!----<NSThread: 0x14649c40>{number = 3, name = (null)}

这样既保证了queue里的task1和task2并发执行，又保证了task3同步执行：必须在task1和task2执行完后开始执行。
![DisPatchGroup](https://upload-images.jianshu.io/upload_images/2427856-4ca74e69a561d366.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# Barrier
顾名思义，Barrier是栅栏的意思，在GCD中起到隔断任务的左右。

```
 func barrier() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label", attributes: .concurrent)
        queue.async {
            sleep(2)
            print("task11!----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task22!----\(Thread.current)")
        }
        queue.async(flags: .barrier) { // 设置flags为barrier
            print("barrier!-----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task33!-----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task44!-----\(Thread.current)")
        }
        print("End")
    }
```

swift的barrier创建与OC使用dispatch_barrier不同，swift是通过传一个barrier的DispatchWorkItemFlags枚举实现的。
> Hello, World!
End
task11!----<NSThread: 0x165caeb0>{number = 2, name = (null)}
task22!----<NSThread: 0x166a5180>{number = 3, name = (null)}
barrier!-----<NSThread: 0x165caeb0>{number = 2, name = (null)}
task44!-----<NSThread: 0x165caeb0>{number = 2, name = (null)}
task33!-----<NSThread: 0x166a5180>{number = 3, name = (null)}

使用barrier确保了task1，task2和task3，task4并发执行，但同时可以确保barrier后的task3,task4在barrier前的task1,task2后面执行。很明显可以看出，barrier和group作用是类似的，就是通过阻塞queue来实现同步功能的，这段代码同样可以使用group实现。但barrier更方便。
![barrier](https://upload-images.jianshu.io/upload_images/2427856-09d60c62cf05af37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
刚才的barrier是通过queue.async(flags: .barrier) {}异步方式创建的，它还有同步方式queue.sync(flags: .barrier) {}（而group只有async方式），两者的区别是什么呢？前面多次提到同步执行需要等待，queue.async(flags: .barrier) {}通过栅栏的形式只会阻塞当前queue这个队列，而如果使用queue.sync(flags: .barrier) {}的方式，那么它还会阻塞主线程，print("End")任务就会在barrier后执行。

```
func barrier() -> Void {
        print("Hello, World!");
        let queue = DispatchQueue.init(label: "queue_label", attributes: .concurrent)
        queue.async {
            sleep(2)
            print("task11!----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task22!----\(Thread.current)")
        }
        queue.sync(flags: .barrier) { // 设置flags为barrier
            print("barrier!-----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task33!-----\(Thread.current)")
        }
        queue.async {
            sleep(2)
            print("task44!-----\(Thread.current)")
        }
        print("End")
    }
```

> Hello, World!
task11!----<NSThread: 0x16e50940>{number = 2, name = (null)}
task22!----<NSThread: 0x16e50ca0>{number = 3, name = (null)}
barrier!-----<NSThread: 0x16e2b1e0>{number = 1, name = main}
End
task33!-----<NSThread: 0x16e50940>{number = 2, name = (null)}
task44!-----<NSThread: 0x16e50ca0>{number = 3, name = (null)}

断断续续的终于把这块写完了，但这里只讲了GCD的一部分，以上也是我个人对GCD的一点理解。如有不对的地方，还请大家多多指教。
