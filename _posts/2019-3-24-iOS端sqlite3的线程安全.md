[sqlite3 多线程和锁 ，优化插入速度及性能优化](http://www.cnblogs.com/huozhong/p/5973938.html)
这篇博客，着重介绍了sqlite3 多线程和锁。这里针对iOS端写个简单的demo:
**验证iOS端sqlite3多线程问题：**

sqlite3 线程模式，在iOS上是2，也就是多线程模式：只要一个数据库连接不被多个线程同时使用就是安全的，多个线程同时使用同一个数据库连接就会有问题了。简单的代码模拟下：
1. 两个线程分别执行读写操作:
```
- (void)creatSqlite{
    FMDatabase * db=[FMDatabase databaseWithPath:_path];
    _db=db;
    if ([_db open]) {
        [_db executeUpdate:@"CREATE TABLE IF NOT EXISTS Student (id integer PRIMARY KEY AUTOINCREMENT, name text NOT NULL);"];
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            for (int i=0; i<10; i++) {
                BOOL s = [_db executeUpdate:@"INSERT INTO Student (name) VALUES (?)",@"小明"];
                NSLog(@"start %d ===success %d",i,s);
            }
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            FMResultSet *set = [_db executeQuery:@"select id from Student"];
            while ([set next]) {
                int vl = [set intForColumn:@"id"];
                NSLog(@"select %d",vl);
            };
        });
    }
}
```
运行试下，果然crash了`（这里使用了三方库FMDB，由于其源码中有容错处理，有这种多线程处理情况只会打印错误log中断操作，所以并不会crash但操作是失败的；这里将其容错判断注释了）`
这是因为：
> prepared statement，它是由数据库连接（的pager）来管理的，使用它也可看成使用这个数据库连接。在多线程模式下，并发对同一个数据库连接调用sqlite3_prepare_v2()来创建prepared statement，或者对同一个数据库连接的任何prepared statement并发调用sqlite3_bind_*()和sqlite3_step()等函数都会出错（在iOS上，该线程会出现EXC_BAD_ACCESS而中止）。这种错误无关读写，就是只读也会出错。

![crash](https://upload-images.jianshu.io/upload_images/2427856-04a2eec2fb950610.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

控制台错误信息：
> BUG IN CLIENT OF sqlite3.dylib: illegal multi-threaded access to database connection

2. 上面是两个线程分别执行了读写操作，再试下同时执行写操作：
```
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            for (int i=0; i<10; i++) {
                BOOL s = [_db executeUpdate:@"INSERT INTO Student (name) VALUES (?)",@"小明"];
                NSLog(@"start1 %d ===success %d",i,s);
            }
        });
        
        for (int i=0; i<10; i++) {
            BOOL s = [_db executeUpdate:@"INSERT INTO Student (name) VALUES (?)",@"小芳"];
            NSLog(@"start2 %d ===success %d",i,s);
        }
```
结果和上面的一样；
3. 同时执行读操作:
```
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            FMResultSet *set = [_db executeQuery:@"select id from Student"];
            while ([set next]) {
                int vl = [set intForColumn:@"id"];
                NSLog(@"select1 %d",vl);
            };
        });
        
        FMResultSet *set = [_db executeQuery:@"select id from Student"];
        while ([set next]) {
            int vl = [set intForColumn:@"id"];
            NSLog(@"select2 %d",vl);
        };
```
结果也是一样;

**为使得所有线程共用全局的数据库连接，可以将sqlite3线程模式更改为串行模式：在初始化SQLite前，调用sqlite3\_config(SQLITE\_CONFIG\_SERIALIZED)启用。**
```
- (BOOL)open {
    if (_db) {
        return YES;
    }
    sqlite3_config(SQLITE_CONFIG_SERIALIZED);
    int err = sqlite3_open([self sqlitePath], (sqlite3**)&_db );
    if(err != SQLITE_OK) {
        NSLog(@"error opening!: %d", err);
        return NO;
    }
    
    if (_maxBusyRetryTimeInterval > 0.0) {
        // set the handler
        [self setMaxBusyRetryTimeInterval:_maxBusyRetryTimeInterval];
    }
    
    
    return YES;
}
```

设置好了后，再看下之前3个的结果，可以正常执行了：
1. 
> 2018-12-15 16:50:48.466926+0800 Test[2527:443214] start 0 ===success 1
2018-12-15 16:50:48.482471+0800 Test[2527:443214] start 1 ===success 1
2018-12-15 16:50:48.494744+0800 Test[2527:443214] start 2 ===success 1
2018-12-15 16:50:48.507210+0800 Test[2527:443214] start 3 ===success 1
2018-12-15 16:50:48.507357+0800 Test[2527:443219] select 1
2018-12-15 16:50:48.523451+0800 Test[2527:443214] start 4 ===success 1
2018-12-15 16:50:48.523487+0800 Test[2527:443219] select 2
2018-12-15 16:50:48.536790+0800 Test[2527:443214] start 5 ===success 1
2018-12-15 16:50:48.536832+0800 Test[2527:443219] select 3
2018-12-15 16:50:48.536972+0800 Test[2527:443219] select 4
2018-12-15 16:50:48.537009+0800 Test[2527:443219] select 5
2018-12-15 16:50:48.537036+0800 Test[2527:443219] select 6
2018-12-15 16:50:48.550073+0800 Test[2527:443214] start 6 ===success 1
2018-12-15 16:50:48.564827+0800 Test[2527:443214] start 7 ===success 1
2018-12-15 16:50:48.582215+0800 Test[2527:443214] start 8 ===success 1
2018-12-15 16:50:48.597302+0800 Test[2527:443214] start 9 ===success 1
2. 
> 2018-12-15 15:21:08.631764+0800 Test[2460:430481] start2 0 ===success 1
2018-12-15 15:21:08.658209+0800 Test[2460:430481] start2 1 ===success 1
2018-12-15 15:21:08.658218+0800 Test[2460:430524] start1 0 ===success 1
2018-12-15 15:21:08.675203+0800 Test[2460:430481] start2 2 ===success 1
2018-12-15 15:21:08.703780+0800 Test[2460:430481] start2 3 ===success 1
2018-12-15 15:21:08.704322+0800 Test[2460:430524] start1 1 ===success 1
2018-12-15 15:21:08.714927+0800 Test[2460:430481] start2 4 ===success 1
2018-12-15 15:21:08.729898+0800 Test[2460:430524] start1 2 ===success 1
2018-12-15 15:21:08.745181+0800 Test[2460:430481] start2 5 ===success 1
2018-12-15 15:21:08.772671+0800 Test[2460:430481] start2 6 ===success 1
2018-12-15 15:21:08.772672+0800 Test[2460:430524] start1 3 ===success 1
2018-12-15 15:21:08.790474+0800 Test[2460:430481] start2 7 ===success 1
2018-12-15 15:21:08.821707+0800 Test[2460:430481] start2 8 ===success 1
2018-12-15 15:21:08.821727+0800 Test[2460:430524] start1 4 ===success 1
2018-12-15 15:21:08.835536+0800 Test[2460:430481] start2 9 ===success 1
2018-12-15 15:21:08.854285+0800 Test[2460:430524] start1 5 ===success 1
2018-12-15 15:21:08.881196+0800 Test[2460:430524] start1 6 ===success 1
2018-12-15 15:21:08.895294+0800 Test[2460:430524] start1 7 ===success 1
2018-12-15 15:21:08.909748+0800 Test[2460:430524] start1 8 ===success 1
2018-12-15 15:21:08.926922+0800 Test[2460:430524] start1 9 ===success 1
3. 
> 2018-12-15 15:12:13.531765+0800 Test[2454:429163] select2 1
2018-12-15 15:12:13.531776+0800 Test[2454:429205] select1 1
2018-12-15 15:12:13.531841+0800 Test[2454:429163] select2 2
2018-12-15 15:12:13.531842+0800 Test[2454:429205] select1 2
2018-12-15 15:12:13.531872+0800 Test[2454:429163] select2 3
2018-12-15 15:12:13.531885+0800 Test[2454:429205] select1 3
2018-12-15 15:12:13.531899+0800 Test[2454:429163] select2 4
2018-12-15 15:12:13.531912+0800 Test[2454:429205] select1 4
2018-12-15 15:12:13.531925+0800 Test[2454:429163] select2 5
2018-12-15 15:12:13.531939+0800 Test[2454:429205] select1 5
2018-12-15 15:12:13.531951+0800 Test[2454:429163] select2 6
2018-12-15 15:12:13.531965+0800 Test[2454:429205] select1 6
2018-12-15 15:12:13.531977+0800 Test[2454:429163] select2 7
2018-12-15 15:12:13.531990+0800 Test[2454:429205] select1 7
2018-12-15 15:12:13.532002+0800 Test[2454:429163] select2 8
2018-12-15 15:12:13.532016+0800 Test[2454:429205] select1 8
2018-12-15 15:12:13.532028+0800 Test[2454:429163] select2 9
2018-12-15 15:12:13.532107+0800 Test[2454:429163] select2 10
2018-12-15 15:12:13.532043+0800 Test[2454:429205] select1 9
2018-12-15 15:12:13.533345+0800 Test[2454:429205] select1 10

**sqlite3多线程模式下，编码实现线程安全：（异步线程下）
要实现线程安全，满足以下一个条件即可：
1. 数据库连接都在一个线程中。
2. 数据库在多个线程都有连接，但不是同时连接。

那么有以下方式编码实现：

- 加锁（确保不是同时连接）
```
        _lock = [[NSLock alloc] init];
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [_lock lock];
            for (int i=0; i<10; i++) {
                BOOL s = [_db executeUpdate:@"INSERT INTO Student (name) VALUES (?)",@"小明"];
                NSLog(@"start %d ===success %d",i,s);
            }
            [_lock unlock];
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [_lock lock];
            FMResultSet *set = [_db executeQuery:@"select id from Student"];
            while ([set next]) {
                int vl = [set intForColumn:@"id"];
                NSLog(@"select %d",vl);
            };
            [_lock unlock];
        });
```
- 串行队列，异步执行：（确保在同一线程中）
```
        dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
        dispatch_async(queue, ^{
            for (int i=0; i<10; i++) {
                BOOL s = [_db executeUpdate:@"INSERT INTO Student (name) VALUES (?)",@"小明"];
                NSLog(@"start %d ===success %d",i,s);
            }
        });
        
        dispatch_async(queue, ^{
            FMResultSet *set = [_db executeQuery:@"select id from Student"];
            while ([set next]) {
                int vl = [set intForColumn:@"id"];
                NSLog(@"select %d",vl);
            };
        });
```
- 使用FMDatabaseQueue，FMDatabaseQueue实现很简单，其实里面就是封装了一个GCD串行队列，队列中任务同步执行达到串行的作用。（确保在同一线程）
```
        FMDatabaseQueue *queue = [[FMDatabaseQueue alloc] initWithPath:_path];
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [queue inDatabase:^(FMDatabase *db) {
                for (int i=0; i<10; i++) {
                    BOOL s = [_db executeUpdate:@"INSERT INTO Student (name) VALUES (?)",@"小明"];
                    NSLog(@"start %d ===success %d",i,s);
                }
            }];
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [queue inDatabase:^(FMDatabase *db) {
                FMResultSet *set = [_db executeQuery:@"select id from Student"];
                while ([set next]) {
                    int vl = [set intForColumn:@"id"];
                    NSLog(@"select %d",vl);
                };
            }];
        });
```

结果都一致
> 2018-12-15 16:53:55.330335+0800 Test[2530:443778] start 0 ===success 1
2018-12-15 16:53:55.345812+0800 Test[2530:443778] start 1 ===success 1
2018-12-15 16:53:55.363252+0800 Test[2530:443778] start 2 ===success 1
2018-12-15 16:53:55.378892+0800 Test[2530:443778] start 3 ===success 1
2018-12-15 16:53:55.396974+0800 Test[2530:443778] start 4 ===success 1
2018-12-15 16:53:55.411347+0800 Test[2530:443778] start 5 ===success 1
2018-12-15 16:53:55.425837+0800 Test[2530:443778] start 6 ===success 1
2018-12-15 16:53:55.442600+0800 Test[2530:443778] start 7 ===success 1
2018-12-15 16:53:55.460784+0800 Test[2530:443778] start 8 ===success 1
2018-12-15 16:53:55.478188+0800 Test[2530:443778] start 9 ===success 1
2018-12-15 16:53:55.480301+0800 Test[2530:443780] select 1
2018-12-15 16:53:55.480416+0800 Test[2530:443780] select 2
2018-12-15 16:53:55.480493+0800 Test[2530:443780] select 3
2018-12-15 16:53:55.480968+0800 Test[2530:443780] select 4
2018-12-15 16:53:55.481118+0800 Test[2530:443780] select 5
2018-12-15 16:53:55.481443+0800 Test[2530:443780] select 6
2018-12-15 16:53:55.481587+0800 Test[2530:443780] select 7
2018-12-15 16:53:55.481679+0800 Test[2530:443780] select 8
2018-12-15 16:53:55.481905+0800 Test[2530:443780] select 9
2018-12-15 16:53:55.481991+0800 Test[2530:443780] select 10
