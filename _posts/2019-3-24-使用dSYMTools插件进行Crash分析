---
layout: default
title: 使用dSYMTools插件进行Crash分析
---

app难免会发生崩溃，debug时发生的崩溃还好说，我们只要设置了All Exceptions断点一般情况都会定位到具体的代码行。但对于发布的版本，用户使用时发生崩溃，问题就没那么简单了。这个时候就需要我们通过解析Crash文件来分析了。关于Crash文件来分析我们首先来看下获取崩溃信息的方式:
- 使用友盟、云测、百度等第三方平台统计
- 连接设备，通过Xcode直接查看设备的崩溃信息
- 通过NSException类获取，上传至自己的服务器
我自己的项目里用的是第三种，这里就主要讲讲通过NSException类获取，并上传至自己的服务器的方式。
#获取crash信息
```
  - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
     NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler);
     return YES;
  }

  // 接收崩溃信息
  void UncaughtExceptionHandler(NSException *exception) {
      // 1.获取Device信息
      NSString *machineName = [Device machineName];
      NSString *systemVersion = [[UIDevice currentDevice] systemVersion];
      // 2.获取NSException信息
      NSArray *symbols = [exception callStackSymbols];
      NSString *reason = [exception reason];
      NSString *name = [exception name];
      // 3.拼接crash信息
      NSString *crash = [NSString stringWithFormat:@"Device:%@ systemVersion:%@<br>%@ Reason:%@<br>ExceptionName:%@",machineName,systemVersion,[symbols componentsJoinedByString:@"<br>"],reason,name];
      // 4.将crash上传至服务器
      .........
  }
```
上传之后在我们的后台就会有crash的记录了:
![crashLog](http://upload-images.jianshu.io/upload_images/2427856-b9f5ef3229918e7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过以上信息我们可以清晰的看出：crash的原因是字典插入了一个nil的对象，但是对于是哪个文件哪个类哪个方法导致的crash我们一无所知。如果只是根据-[__NSPlaceholderDictionary initWithObjects:forKeys:count:]: attempt to insert nil object from objects[0]这个错误类型去代码里查找，那无异于大海捞针。那我们如何根据以上信息准确定位crash的代码呢？
#crash分析
网上关于crash分析的资料有很多，这里我主要分享下如何通过dSYMTools插件分析crash。
- 获取dSYM 符号集
符号集是我们对APP进行打包之后，和.app文件同级的后缀名为.dSYM的文件。
可以通过Xcode->Window->Organizer->选择archive包->Show in Finder获得.xcarchive文件。进行崩溃信息符号化的时候，必须使用当前应用打包的电脑所生成的dSYM文件，其他电脑生成的文件可能会导致分析不准确的问题。
- 下载[dSYMTools插件](https://github.com/answer-huang/dSYMTools)，通过Xcode运行，界面如下:
(“请选择文件名”一栏能自动识别在我们电脑里的所有.xcarchive.)
![dSYMTools界面](http://upload-images.jianshu.io/upload_images/2427856-9c4155bd99351936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 根据自己项目实际情况选择要分析的.xcarchive文件以及CPU类型。选择后UUID会自动生成。
- 填入Slide Address以及错误信息内存地址
Slide Address填写上图crashLog中标了红1的0x0000000104ab9610，而错误信息内存地址一栏是要经过计算的，这步比较重要这里着重讲下计算的过程：
1.将上图crashLog中红1的0x0000000104ab9610转换为10进制
2.将转换后的值加上红2的3626512得到结果再转换为16进制，算出错误信息内存地址：0x104E2EC20填入输入框
（为什么要这样计算？其实这与crash内存地址偏移有关，关于这块我也不是很了解，大家可以百度下）
- 点击分析，结果出来了并能精确定位到具体某行代码
![分析结果](http://upload-images.jianshu.io/upload_images/2427856-1cf2650a027f8a00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 最后可以轻松加愉快的找到ChatViewController第535行去修复这个bug了


