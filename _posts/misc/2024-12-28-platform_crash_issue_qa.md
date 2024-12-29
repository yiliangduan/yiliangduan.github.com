---
title: 'Unity 应用Crash问题解决方法'
date: 2024-12-28 21:32:25
type: "tags"
tags: unity
comments: false
---

最近在处理Crash问题总结了一些心得这里记录下。

### Crash问题处理流程

Unity App Crash问题主要分为几类：

* 系统资源（包括内存，CPU等）
* 业务逻辑（主要是死循环这类，因为是C#写的没有空指针这种问题，Unity为数不多的有点）
* 引擎自己的代码逻辑（空指针，死循环等）
* 第三方SDK造成（这类最难查）
* 其他原因

针对不同平台，我们定位Crash需要的数据信息也不一样：

* Android： adb logcat日志 + so符号文件
* iOS：ips日志 + dSYM符号文件
* Window：dump日志 + pdb符号文件

可以看到无论哪个平台都需要符号文件，所以如果是标准化流程的话，构建包的流水线每次都应该生成符号文件，并且上传到云服务器，保证每个包都能取到对应的符号文件。另外这些平台都需要提供崩溃日志，但是如果是外网玩家出现的取不到日志的情况呢？一个实用的Crash工具[CrashSight](https://crashsight.qq.com/)，App继承了CrashSight之后能够捕获到发生的Crash日志并且上报的服务器，我们只需要把包的符号文件上传到CrashSight服务器，CrashSight能够把每个上报的Crash堆栈自动找到对应的符号文件还原成符号化的堆栈。整个过程是这样的：

![](/images/crash/crashsight_flow.png)

没有用CrashSight的可以按照这个流程来构建一个自己项目的解决Crash问题工作流。另外客户端发生的Crash问题 CrashSight 并不是每次都能捕获到的，其他工具也是如此。这种情况我们只能通过内存的适配和功能测试，有案例发生时，我们拿到日志文件首先符号化来还原堆栈，并且定位问题。CrashSight用户端的页面提供了一个符号化的工具，没有接入CrashSight的话我们也可以用自己做的工具。

说完了整体处理流程，再看下一些具体的Crash处理经验。

### 常见Crash问题具体案例

#### oom 

oom（out of memory）和字面意思一样，就是系统内存不够或者给定当前的app的内存超出上限了，然后分配不到内存Crash了。

iOS系统有Jetsam机制，当内存不够时Jetsam会终止App，并且会生成Jetsam的ips日志，日志名字一般为  *JetsamEventxxx.ips*。所以当Crash时我们发现系统生成了Jetsam ips日志时，基本上可以判断是由于oom造成的。Jetsam日志里面会记录下当前App所有进程分配的内存情况，日志文件里我们可以看到如下内容（省略了很多内容的）：
```c#
{"bug_type":"298","timestamp":"2020-03-19 17:30:28.39 +0800","os_version":"iPhone OS 13.3.1 (17D50)","incident_id":"7F111601-BC7A-4BD7-F468-CE3370053097"}
{
  "product" : "iPhone12,3",
  "pageSize" : 16384, //每个page占用的字节数
},
  "largestProcess" : "XXX(App的BundleName)",
  "genCounter" : 0,
  "processes" : [
  {
    "uuid" : "2dd5eb1e-fd31-36c2-99d9-bcbff44efbb9",
    "rpages" : 164, //占用的内存数 rpages * pageSize
  },
```

算出来内存大小可以帮助反馈App的内存性能状况，来做内存优化的参考。

Android系统在App内存过高时终止进程也会输出日志信息(Android系统各个厂商都有自己的定制，不一样每台手机每次都会生成，遇到过低端设备疑似没有生成的情况)，Android系统的日志是直接输出的系统的调试Log中，我们可以通过 adb工具拿到。存储系统的日志本身是个循环利用的一块内存，我们需要在Crash发生之后就拿日志，否则系统如果继续运行最后存储Crash日志的那块内存会被重复利用，日志信息会被覆盖。

所以当测试同学遇到Crash问题，第一个动作就是通过adb工具获取log。几个常用的adb命令行：

```shell
//清理缓存。如果是专门的Crash测试，可以先执行清楚日志缓存信息再测试
adb logcat -c

//获取日志，并且输出到xxx_crash.log文件中
adb logcat -v time > xxx_carsh.log
```

拿到日志之后，可以直接在日志里面搜索关键字 *lowmemorykiller* 或者 *kill*，找到自己App的包名：

![low_memory_killer](/images/crash/low_memory_killer.png)

如果能找到自己包名的 *lowmemorykiller* (不同设备可能这个关键字不一样)可以表明就是oom问题了。

ANR

ANR（Android Not Responding）问题是安卓的特有问题，Android系统中，*ActivityManagerServer(简称AMS) 和WindowManagerServie(简称WMS)*会检测App的响应时间，如果App在特定时间无法响应屏幕触摸或者键盘输入，或者特定事件没有处理完毕，就会触发ANR。

出现以下任何情况时，系统都会针对您的应用触发 ANR：
- **Input dispatching timed out**：如果您的应用在 5 秒内未响应输入事件（例如按键或屏幕触摸）。
- **Executing service:**：如果应用声明的服务无法在几秒内完成 `Service.onCreate()` 和 `Service.onStartCommand()`/`Service.onBind()` 执行。
- **Service.startForeground() not called**：如果您的应用使用 `Context.startForegroundService()` 在前台启动新服务，但该服务在 5 秒内未调用 `startForeground()`。
- **Broadcast of intent**：如果 [`BroadcastReceiver`](https://developer.android.com/reference/android/content/BroadcastReceiver?hl=zh-cn) 在设定的一段时间内没有执行完毕。如果应用有任何前台 activity，此超时期限为 5 秒。
- **JobScheduler interactions**：如果 [`JobService`](https://developer.android.com/reference/android/app/job/JobService?hl=zh-cn) 未在几秒钟内从 `JobService.onStartJob()` 或 `JobService.onStopJob()` 返回，或者如果[user-initiaed job](https://developer.android.com/reference/android/app/job/JobParameters?hl=zh-cn#isUserInitiatedJob())启动，而您的应用在调用 `JobService.onStartJob()` 后的几秒内未调用 `JobService.setNotification()`。对于以 Android 13 及更低版本为目标平台的应用，ANR 会保持静默状态，且不会报告给应用。对于以 Android 14 及更高版本为目标平台的应用，ANR 会保持活动状态，并会报告给应用。

发生ANR拿到logcat日志之后，可以在日志里面看到：

![anr](/images/crash/anr.jpg)

这里是5秒钟未响应事件，对于Unity的App来说其实没有给到明确的原因，发生这种情况一般是游戏的线程卡住了，导致未响应。此时我们需要根据Unity日志里面匹配时间点来看我们在处理什么逻辑导致卡住的（或者是App里面接入的SDK卡住的）。

#### 业务逻辑

业务逻辑造成的，有日志的话看Crash堆栈可以定位到具体代码，这个方式很明确，不需要多讲。



---

引用：

1. [iOS 内存 Jetsam 机制探究](https://juejin.cn/post/6844903508848689166)

2. [带你打造一套 APM 监控系统 之 OOM 问题](https://cloud.tencent.com/developer/inventory/513/article/1662232)
3. [Logcat命令行工具](https://developer.android.com/tools/logcat?hl=zh-cn)
4. [Android ANR：原理分析及解决办法](https://juejin.cn/post/7018172565369651230)
