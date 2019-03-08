---
title: Android性能优化 - 内存优化
date: 2019-03-08 15:34:26
tags:
 - 性能优化
 - 内存优化
---

在内存方面我们通常会遇到几个问题：内存空间占用过大、内存泄露、内存溢出、内存抖动等。



<!-- more-->

## 1. 为什么要做内存优化

* 避免OOM导致应用崩溃，提高应用稳定性

* 内存资源是有限的，避免系统内存不足导致LowMemoryKiller机制触发杀死应用



### 1.1 OOM是什么?

Android中定义了一个进程的内存上限，超过之后便会OOM崩溃。

上限值可通过系统属性查看：

![](https://i.loli.net/2019/03/08/5c821d02c5790.png)

第一个属性为一个标准应用能申请的最大内存。

第二个属性为应用添加**largeHeap=true**之后能够申请的最大内存。



### 1.2 LowMemoryKiller机制

####进程优先级

当系统空间不足时，会按照进程的优先级对进程进行回收来腾出更多的空间给有需要的进程使用。

Android中进程按优先级大概分为几个类型，优先级从高到底依次为：

![](https://i.loli.net/2019/03/08/5c821da6d4587.png)

**前台进程**：正在与用户交互，Activity Service BroadcastReceiver正在交互。Service

是在前台运行的

**可见进程**：onPause状态的Activity 或 进程持有的Service与前台或者可见Activity绑定

**服务进程**：进程运行着一个不可见的Service，比如后台播放音乐等

**后台进程**：Home键退到后台时，进程持有一个stop的Activity。

**空进程**：一个进程不包含任何活跃的组件，缓存下来为了下次更快地启动。



上面只是一个大概的分类，ActivityManagerService会对进程进行评分，确定一个oom_adj值，进程回收时LowerMemoryKiller会根据这个评分进行判断

当内存达到阈值时，会根据oom_adj和进程所占内存的大小来杀死进程。

oom_adj相同的话，内存占用大的进程会优先被杀死。



#### 保活方法

为了提高应用打开的速度，或者保证自己的核心应用不被杀死（比如OSService）。

按照策略：

**提高进程优先级，保证不被杀死**

1、开启前台service，比如播放音乐的通知栏

2、利用系统漏洞开启前台service（不显示通知）

```java
 @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (Build.VERSION.SDK_INT < 18) {
            startForeground(GRAY_SERVICE_ID, new Notification());//API < 18 ，此方法能有效隐藏Notification上的图标
        } else {
            Intent innerIntent = new Intent(this, GrayInnerService.class);
            startService(innerIntent);
            startForeground(GRAY_SERVICE_ID, new Notification());
        }

        return super.onStartCommand(intent, flags, startId);
    }
```



3、1像素Activity

4、系统应用添加persist=true属性

5、内存优化，减少内存占用



**应用被杀死之后，重新拉活**

1、Service#onStartCommand返回STICKY。

2、监听广播拉活（系统广播、应用全家桶、推送服务等）

3、双进程守护

4、JobScheduler（5.0+）

...



## 2. 常见的内存问题

###2.1 GC机制
![](https://i.loli.net/2019/03/08/5c821e6d2bf4c.png)

JVM中从GC Roots对象开始遍历进行可达性分析，对于不可达的对象在GC时被标记回收。

在Java语言里，可作为GC Roots对象的包括如下几种：

a.虚拟机栈(栈桢中的本地变量表)中的引用的对象

b.方法区中的类静态属性引用的对象

c.方法区中的常量引用的对象

d.本地方法栈中JNI的引用的对象 

摘自《深入理解Java虚拟机》

### 2.2 内存泄露

导致OOM的最主要原因。

内存泄露是指进程分配的内存由于某些原因无法被释放，一般是长生命周期的对象强引用了短生命周期的对象。

**常见内存泄露原因**：

* 单例：引用了生命周期较短的对象

* 静态变量

* 非静态内部类和匿名内部类（引用外部类对象）

* 资源使用后未释放

  IO、Cursor等资源使用后要及时关闭，释放缓冲对象、
  广播注册，取消注册成对出现、
  WebView用完之后destory

* 异步操作在界面退出后没有停止

  MVP架构P层进行异步操作，退出时没有释放V层

### 2.3 内存占用过大

 * 图片的使用（最主要的原因）

> Q: 一张20KB的png图片放在ImageView上占用多大的内存空间?
>
> A: 跟图片在apk中的资源文件夹dpi和目标设备dpi有关，跟ImageView的大小和png图片本身的大小无关。
>
> 公式为: widthPx * heightPx  * targetDpi / inDpi * targetDpi/inDpi * 颜色深度
>
> 详见: [你的 Bitmap 究竟占多大内存？](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=403263974&idx=1&sn=b0315addbc47f3c38e65d9c633a12cd6&scene=21#wechat_redirect)

优化方案有:

1. 使用合适尺寸的图片或者调整采样率（图片比View大时）

2. 图片格式RGB_555(没有透明通道）

3. loading动画尽量用代码实现，减少图片

4. Bitmap复用（参考Glide的BitmapPool实现）

* 一些数据结构的优化

  ArrayMap、SparseArray等Android提供的数据接口来替代HashMap、
  使用注解代替枚举类型等

* 不好的代码

  使用ListView没有复用convertView、
  提前实例化用不到的对象

### 2.4 内存抖动

内存短时间内大量地分配和回收，频繁的GC可能会导致卡顿问题。甚至会导致OOM。

优化方案有：

1、尽量避免在循环体内大量创建对象，应该把对象创建移到循环体外。

2、自定义View的onDraw()方法会被频繁调用，不应该频繁的创建对象。

3、使用对象池（比如MessagePool等）



**监控和优化方案：**

LeakCanaray //TODO 实现原理

Android profiler

实战如何看内存信息



MAT







### 内存占用过大

解决了内存泄露问题之后，开始着手对应用正常分配的内存进行优化，减小基数

**常见原因**

* 图片

图片到底占了多大？

跟放置的文件夹和目标density有关： widthPx * heightPx * 颜色深度。 与ImageView大小无关。



优化：

1、采样率

2、图片格式RGB_565

3、loading动画自己画 //TODO svg等

4、缓存池，系统级别的缓存、软引用、弱引用

5、及时回收，recycler



Bitmap的优化、缓存池的实现。//TODO参考第三方库的实现



* 数据结构优化

ArrayMap、SparseArray等Android提供的数据接口来替代HashMap等。

枚举：使用注解代替

ListView的convertView复用

懒加载（不需要的对象延迟加载）

**数据相关**：序列化数据使用protobuf可以比xml省30%内存，慎用shareprefercnce，因为对于同一个sp，会将整个xml文件载入内存，有时候为了读一个配置，就会将几百k的数据读进内存，数据库字段尽量精简，只读取所需字段。



### 内存抖动

内存短时间内大量地分配和回收，频繁的GC可能会导致卡顿问题。甚至会导致OOM

主要原因还是有因为大量小的对象频繁创建，导致内存碎片，从而当需要分配内存时，虽然总体上还是有剩余内存可分配，而由于这些内存不连续，导致无法分配，系统直接就返回OOM了



## 3. 内存检测工具的使用

### 3.1 LeakCanary

内存泄露问题检测的利器，使用起来非常简单。

build.gradle

```java
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
  releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
  // Optional, if you use support library fragments:
  debugImplementation 'com.squareup.leakcanary:leakcanary-support-fragment:1.6.3'
}
```

Application中

```java
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
    // Normal app init code...
  }
}
```



当应用中有内存泄露问题时，会有如下界面，进行引用的分析：

![](https://i.loli.net/2019/03/08/5c822278b7645.png)



**LeakCanary是怎么来检测内存泄露的呢？**

十分简单，两步：

1、监听Activity的onDestroy

![](https://i.loli.net/2019/03/08/5c8222f314c14.png)

Application提供了callback可以监听应用中任何一个Activity的生命周期变化，这里在Activity destroy时对Activity进行watch。

2、利用弱引用+引用队列检测对象是否被回收。

> 这里说一下弱引用的特性：
>
> 1、被弱引用引用的对象，在GC的时候一定会被回收。
>
> 2、弱引用创建时可以传入一个引用队列，如果对象被回收，则这个弱引用会被放入引用队列中。
>
> 由此，检测一个对象是否被回收就通过弱引用来引用，然后触发GC，最后检测引用队列中是否存在这个引用即可。

![](https://i.loli.net/2019/03/08/5c8223a9954e1.png)



上面这种简单的使用只能检测出Activity对象的泄露，如果要检测其他对象的泄露，可以代码中主动watch想要检测的对象，这里不细说。

### 3.2 Android Profiler工具

AS中Run -> Profile，可以查看CPU、内存、网络等信息。



![](https://i.loli.net/2019/03/08/5c8224d439e36.png)

检查内存泄露的方式为：

1、反复多几次进入、退出怀疑的界面

2、最后退出怀疑的界面之后，手动触发一下GC

3、dump现在内存信息（上图），进行分析。

Android Profiler工具已经可以提供比较详细的信息，比如引用信息等，个人感觉已经可以替代老牌内存分析工具MAT了。这里不详细介绍工具的用法，大家可以实际去操作一遍就比较容易理解。



### 3.3 MAT工具

![](https://i.loli.net/2019/03/08/5c8225f67dd5d.png)



Android Profiler工具可以将内存信息导出为hprof文件，然后必须利用platform-tools中提供的工具转换成MAT能够识别的hprof文件，然后打开进行分析：

```shell
hprof-conv leak.hprof leak-mat.hprof
```

MAT工具也提供了大量的信息供开发者进行分析，大家实际操作一下分析，更加容易理解。



## 总结

1、对内存优化可以避免OOM，或者被系统杀死，提高系统稳定性。

2、介绍了常见的内存问题的优化手段：内存泄露、内存溢出、内存抖动等。

3、介绍了三个检测、分析内存的工具：LeakCanary、Android Profiler、MAT。只要动手实际操作几遍，就可以比较熟练地掌握这几个工具的使用