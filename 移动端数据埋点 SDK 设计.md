##### 前言

移动应用在正式上架之后，程序员关心的是应用运行的稳定型，而产品与运营的同学关注的是对于用户行为的收集与调研。这个过程需要记录应用运行时的程序状态，以及用户的行为。许多的应用会通过集成市场的第三方 SDK（友盟、小米等）完成相关的统计任务。但是许多应用为了数据的安全性等考虑，会打造自己的数据统计 SDK。本文记录一下自己如何设计与实现的一个移动端数据统计 SDK。

##### 数据统计技术分析

目前客户端埋点技术可以根据对埋点数据的来源分为以下三种类型：代码埋点、可视化埋点、无痕埋点。

（1）代码埋点，代码埋点出现的最早，也是最主流、通用的埋点技术。它主要通过程序员在开发阶段在嵌入埋点代码。然后在应用启动的时候初始化 SDK，最后根据相关策略上传数据置服务器。

（2）可视化埋点，利用可交互手段来替代程序员手动埋点的技术。如果感兴趣可以了解下 mixpanel，此外它对它的源码进行了开源。

（3）无痕埋点，无痕埋点是通过 AOP 等技术手段在程序运行时尽可能的收集更多的数据，然后上传给服务器。

每种埋点技术都有着它自己的优点与缺点，本次实现的数据统计 SDK 是通过代码埋点。程序运行时通过收集预先设置的埋点信息，然后再根据发送策略、应用状态等信息及时的上传数据置服务端。

##### 需求介绍

在开发前，先整理下 SDK 所需要实现的功能。

（1）数据上传需要有不同的上传策略：（1）实时上传（2）时间间隔上传（3）每天上传

（2）具有数据缓存能力，例如当用户没有打开网络请求权限的时候，会将数据缓存到本地磁盘中。

（3）传输数据合并，例如添加用户信息，相同类型数据合并，减少网络请求数据。

（4）上传策略可以由服务端配置

##### 整体结构设计

数据埋点统计具有频繁性的特别，在各个页面上都会有统计的需求。如果收到数据后立即进行处理，会占用应用主线程的资源，例如网络请求。而且导致需要特别频繁的发起请求。因此自己在这个过程当中设计了一个数据统计 SDK 。利用定时周期任务处理一个时间间隔内的数据，有限的减少网络请求的次数。并且方便扩展不同的发送策略，只需要结合策略将事件放置到不同打队列当中。

![](https://github.com/yuhuibin123/imageDB/blob/master/image_article_count.png?raw=true)

##### 具体实现

1. 初始化 SDK。在应用启动的时候，调用初始化接口。主要做的工作，（1）利用 ScheduleExecutorService定时周期的执行进行数据统计处理工作（2）初始化 SDK  数据，主要进行恢复本地磁盘数据缓存。

```java
public synchronized void init(final Context context,
                                  final String channel, final String uuid, final String account) {
        // 开启定时检查线程
        mTimerService.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
         		// 定时处理任务
                if (mInit) {
                    onTimer();
                }
                else {
                // 初始化 SDK  
                    initSDK(channel, uuid, account);
                }
            }
        }, TIMER_START_IN_SECONDS, TIMER_DELAY_IN_SECONDS, TimeUnit.SECONDS);
    }

```

在初始化 SDK 数据，这部分主要工作：（1）初始化默认参数，因为每个上传的数据都需要添加用户信息等默认参数（2）将本地磁盘缓存的数据放置到发送队列。（3）读取服务器配置

```
private void initSDK(final String channel, final String uuid, final String account) {
    if (WirelessZProtocol.INSTANCE().getApplicationContext() == null) {
        throw new IllegalArgumentException("WZP not init");
    }

    //初始化默认参数
    DeviceInfo deviceInfo = new DeviceInfo(mContext);
    mNCollectorManager.getDataManager().setUserParams(deviceInfo.getUserParams());
    
    // 每天首次打开时，将每日数据从SP转移到等待队列
    if (NCollectorManager.getInstance(mContext).getDataManager().isFirstOpenToday()) {
        UploadInterface uploadPriority = new UploadDaily(mContext);
        uploadPriority.moveToSendCache();
    }

    // 把本地发送失败数据，从缓存中转移到等待队列
    mNCollectorManager.getDataManager().getErrorQueueFromSp();

    // 从服务器读取配置信息表
    UploadPriorityConfig mConfig = new UploadPriorityConfig(mContext);
    mConfig.readPriorityFromServer();
}
```

2. 在用户使用的过程当中会根据埋点数据事实的传递到 SDK 中，定时周期性的任务会对埋点数据进行处理，根据发送策略定时的放置到对应的处理队列中。

```
private synchronized void onTimer() {
    // 将消息放置到对应的待处理队列中
    mNCollectorManager.getDataManager().moveToWaitingQueue();

    if (NCollectorUtil.isNetworkAvailable(mContext)) {
    //  如果能够进行网络通信，则将消息移动到待发送队列，
        mNCollectorManager.getDataManager().moveToSendingQueue();
        mNCollectorManager.getUpload().uploadLocalData();
    }
    // 将需要缓存数据存储到内存中
    mNCollectorManager.getDataManager().saveDailyQueueToSP();
    mNCollectorManager.getDataManager().saveErrorQueueToSP();
}
```

（1）首先，先介绍下 SDK 运行的时候，用到的几种的队列：元数据队列、错误数据队列、发送队列，上传策略队列。

每次数据发送给 SDK 的时候都会立即放入到元数据队列中，等待定时任务来处理数据。

```java
public void addToMetadataQueue(String key, Object value) {
	// 元数据
    Metadata metadata = new Metadata(key, value);
    getMetadataQueue().add(metadata);
}
```

（2）定时任务运行的时候，首先会对元数据队列进行一一处理。根据数据的 key 来选择对应的传输策略进行数据处理。例如需要立即上传的数据，会立刻转存到待处理队列中。这里介绍下传输策略的实现，定义了一个接口，包括了两个方法：将数据保存到对应策略的缓存中与将不同发送策略的数据转存到发送队列中；

```java
public interface UploadInterface {
    // 将数据保存到对应策略的缓存中
    void storeToCache(final String eventName, final Object object);
    
    //将不同发送策略的数据转存到发送队列中
    void moveToSendCache();
}
```

以立即上传为例介绍下上传策略的实现：

```java
public class UploadInstant extends UploadPriority implements UploadInterface {

    @Override
    public void moveToSendCache() {
        // 移动到待处理队列中
        storeToWaitingCache(mInstantDailyList);
        mInstantDailyList.clear();
    }

    List mInstantDailyList = new ArrayList();

    @Override
    public void storeToCache(String eventName, Object object) {
        HashMap<String, Object> event = new HashMap<>();
        event.put(eventName, object);
        // 将数据暂存
        mInstantDailyList.add(event);
    }
}
```

然后在处理元数据的时候一一遍历调用适合的策略进行处理。

```java
   public void moveToWaitingQueue() {
       for (Metadata metadata : getMetadataQueue()) {
       		UploadInterface uploadPriority = mConfig.getUploadPriority(metadata.getKey());
       		uploadPriority.storeToCache(metadata.getKey(), metadata.getValue());
       }
        mConfig.getUploadInstant().moveToSendCache();
    }
```

（3）上传数据，在处理完元数据后，如果符合添加会将待处理的数据转存到发送队列，将数据拼接成字符串后通过网络请求接口上传数据。

```
private void sendByTracing(String data) {
   // 调用 WZP 的接口上传数据
   WZPUnit u = Tracing.INSTANCE().remoteMessage(null, ILogger.LogLevel.REMOTE, mUserName, data);
   if (u.getResponseCode() != 200) {
   		// 上传失败时又重新存到待处理队列中。
         moveToWaitingQueue(data);
   }
}
```

（4）上传数据结束后，会将发送失败的数据以及其他数据保存到磁盘中。

##### 总结

在完成 Android SDK 的实现任务后，学习了 iOS 开发的基础知识，利用 OC 实现了相同设计的 SDK。在这个过程收获最大的是从零开始构建一个系统，考虑系统运行的过程当中可能存在问题，以及系统运行的健壮性。此外，学习 iOS 开发，给自己打开了新的世界。了解到 iOS 的基础语言知识、多线程管理、以及 UI 等知识。