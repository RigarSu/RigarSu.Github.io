---
title: Android内存泄漏和检查——结合项目实例解析
date: 2019-04-19 21:28:31
tags:
- 内存泄漏
categories:
- 性能优化

---

### 前言
在我们版本迭代的过程中，内存泄漏是我们时刻关注，但又经常忽略的烦人问题。几乎每个大版本迭代都会出现新的内存泄漏点，在版本开发阶段经常被忽略，直到灰度才被发现，内存泄漏导致APP内存紧张，导致整体卡顿，OOM崩溃，ANR等问题，这里总结了一些内存泄漏的常见点和检查方法。
### 一、什么是内存泄漏？
1. 内存的自动回收
java中，对象是通过引用与其关联的，如果一个对象没有任何引用，那么这个对象就被认为是”无用“的，java的垃圾回收机制，即是使用引用分析法，对象引用类似一个有向图，选定一系列的索引起点，若某个对象与其均无可达路径，则该对象会被标记为回收对象，在垃圾回收时自动回收。
2. java内存泄漏
如果程序中存在一些对象，这些对象有两个特征，一、在对象引用有向图中，这些对象存在可达路径；二、应用程序不再使用这些对象；那么这些对象则无法被垃圾回收器回收，且不会再用到，这即是java中的内存泄漏。


<!-- more -->


### 二、Android常见的内存泄漏
1. Context被持有造成View泄漏，一般是Context被View以外的组件持有引用，如Model，工作线程等，导致在界面销毁是Context仍被引用，从而造成泄漏。如：
```
public class AppModel {
    private static AppModel instance;
    private Context context;
    private AppModel() {
    }
    public static AppModel getInstance(Context context) {
        if (instance != null) {
            instance = new AppModel(context);
        }
        return instance;
    }
    
    public doSomething(Context ctx) {
        context = ctx;
        //do something
    }
}
```
解决方案：在规范开发中并不常见，应该在团队中规范开发规则，Context应该尽量不在界面以外的组件使用，必要是可以用Application的Context，或者其他解决方案。


2. 非静态内部类持有外部类引用
```
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        switch (msg.what) {
            case SHOW_DIALOG:
                showPleaseDialog();
                break;
            case DISMISS_DIALOG:
                dismissDialog();
                break;
        }
    }
};
```
如果这样写，lint会检查并提示：Handler classes should be static or leaks might occur。
Handler是非静态内部类，会持有外部类的引用，Handler如果是跟主线程的Looper绑定，那么如果界面销毁时，发到主线程的消息，仍未被处理，那么该消息持有Handler的引用，而Handler持有界面引用，从而导致泄漏。
解决方案：建议不在View层做业务处理，View只做刷新职责，在ViewModel处理好相关业务，并通过LiveData或广播等通知View刷新，保证View层不泄漏，ViewModel层使用静态内部类，或者使用其他更优雅的解决方案。

3. 匿名内部类持有外部类的引用。
```
mHandler.postDelayed(new Runnable() {
    @Override
    public void run() {
    }
}, 5000);
new Thread(new Runnable() {
    @Override
    public void run() {
        SystemClock.sleep(10000);
    }
}).start();
```
各种Runnable、Callback等，匿名内部类写法比较简洁，也带来泄漏的隐患。内部类的写法不跨越生命周期的话，是没有问题的，如果在更长的生命周期的对象中使用，则发生泄漏。异步线程等全局对象持有匿名内部类，而匿名内部类持有外部类引用，造成泄漏。
解决方案：同2。同时，如果有异步任务，大部分场景需要在界面销毁时停止，可以统一使用带生命周期异步任务，在界面销毁时终止任务。可以方便的使用界面的Lifecycle监听界面状态。

4. 监听器没有注销导致的内存泄漏

比较常见的内存泄漏点，开发者在无意中经常漏掉注销监听，导致界面等在全局监听器中无法释放，从而内存泄漏。
解决方案：在界面带有生命周期等控件中，通过Lifecycle监听其状态变化，在销毁时自动注销监听器。或者在父类统一处理注册和注销监听器。
```
fun Lifecycle.subScribe(subscribBlock: () -> Unit, unSubscribBlock: () -> Unit) {
    subscribBlock()
    addObserver(object : LifecycleObserver {
        @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
        fun destroy() {
            unSubscribBlock()
        }
    })
}
```
### 三、内存泄漏的检查
1. LeakCanary

Github: [LeakCanary](https://github.com/square/leakcanary)
LeakCanary will automatically show a notification when an activity or support fragment memory leak is detected in your debug build.
众所周知，检查内存泄漏的常用工具，使用也很简单，原理是在Application注册并监听Activity的生命周期，以此来检查见面的内存泄漏，LeakCanary可以检查到一些明显的内存泄漏点，但一些非界面组件，或者隐晦的泄漏点难以发现，因此要结合更加细致的工具检查。

2. Android Studio自带的工具Profile

官方文档：[使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler)
Memory Profile是个方便易用的工具，具体使用方法文档中已经详细描述了。但是它缺少可以辅助分析的工具（而且一旦将内存dump下来开始分析，AS就疯狂卡顿甚至还ANR，不知道是不是我的问题），建议在操作完需要检查的程序之后，GC并Dump下内存，保存后用Memoroy Analyzer分析。

### 四、如何使用Memory Analyzer Tool（MAT）检查内存？
1. 将Profile下载下来的hprof文件使用**android_sdk/platform-tools**的工具专为标准的Java SE hprof文件：**hprof-conv heap-original.hprof heap-converted.hprof**


2. 此时可以用MAT打开该文件，初始界面如下：
{% asset_img 初始界面.png 初始界面 %}
可以看到有很多Action可以选择，每个Action都有相应的说明，这里简单介绍常用的几种：
Action：
Histogram：应用所分配的对象和实例数
Dominator Tree: 依赖树，后面再说
Top Consumers: 内存占用最大的对象
Duplicate Classes: 重复加载的类
Reports:
Leak Suspects：可能的内存泄漏点
Top Components: 堆占比大于1%的组件
Step By Step:
Component Report: 一步步教你分析的教程


3. 点击Histogram，展示所有对象和实例，第一行支持正则表达式的搜索，其中有几个概念需要认识一下：

Objects: 顾名思义，实例数量
Shallow Heap: 一个对象结构占用的内存大小（对象本身占用的内存，不包括其内部引用对象大小）
Retained Heap: 一个对象所能访问到的所有对象的浅堆之和，即是对象被回收后，能释放的真实内存空间
如下图：
（1）可以简单筛选，看看Activity的存活情况，并可以按以上三个属性升降排序，如果该界面已经销毁一段时间，仍然常驻内存，则可能发生了泄漏。以此类推，你可以有一些猜想对象，并在此检查是否符合预期。
（2）如果发生内存泄漏，可以右键该对象，Merge Shortest Paths to GC Roots，找到它的强引用链（其他引用该对象会被GC清理），分析是哪里对它有引用造成的泄漏。
（3）这里右键还可以查看对象的List Object -> with outgonging references，当前对象引用的对象，以及with ingonging references引用该对象的对象。等等该对象相关的属性和引用。
（4）Window导航栏还有一个Inspector可以查看选择对象的各种属性，很强大，这里不展开。
（5）导航栏Thread Overview可以看到当前线程的状态信息。
{% asset_img Histogram.png Histogram %}


4. Leak Suspects，打开之后它会帮你分析一些可能泄漏的对象，再根据自己分析，结合以上工具，分析是否是真的泄漏点。


5. Dominator Tree，也是比较常用的工具，先简单描述一下支配树。

{% asset_img 引用关系与支配树.png 引用关系与支配树 %}
支配树描述了对象实例间的支配关系，在对象引用图中，所有经过B的路径都经过A，那么A则支配B。
看一下官方的一个例子，左边是对象引用图，右边是其对应的支配树。
- 对象A和B由根对象直接支配
- 经过对象C的路径可以经过A和B，则C也由根对象直接支配
- 所有经过H的路径都经过C，则H由C支配
- 支配树中顶点A的子树，则为对象A的深堆

以此类推，可以从左边得到右边的支配图，由支配图可以更加直观看到对象的深堆（Retained Heap）和内存释放链。如果依赖树中出现了不该出现的对象，基本可以认为其发生了内存泄漏，同样可以右键，Merge Shortest Paths to GC Roots，查看其强引用链。依赖树中的大对象常常也是观察疑似内存泄漏的点，值得仔细排查。

### 五、项目实例解析
#### 例一：
操作步骤：在某个界面重复退出重进，反复几次后，退出该界面，回到首页，等待一分钟左右，触发GC，dump下内存，用MAT打开。
检查方法：在Histogram中搜索该界面，发现有两个实例，Merge Shortest Paths to GC Roots，查看其强引用链，结果如下：
{% asset_img 实例一，周星入口泄漏.png 实例一，周星入口泄漏 %}
结果分析：可以看到，该界面忍然有两个被强引用，都是WeekStarBanner被RxBus引用，即是注册了RxBus没有注销通知。
代码分析：检查到该View在构造函数注册RxBus的通知，而在onDetachedFromWindow注销通知，那么也就是说View创建了，初始化注册了通知，但是onDetachedFromWindow没有回掉，为什么？原来该View在Activity创建就会初始化，但是要网络请求，查是否开放才决定是否显示，这是如果查询结果没返回而退出界面，这个时候View没有attch to window，所以onDetachedFromWindow回调自然也不会掉用，导致泄漏。
解决方法：注册和注销，最好在View对应的生命周期调用，分别在onAttachedToWindow和onDetachedFromWindow注册和注销，可解决该问题。或者设计带有生命周期的注册器，自动在生命周期结束时自动注销。

#### 例二：
操作步骤：打开APP，无其他操作，在首页等待一段后手动触发GC，并dump内存，用MAT打开。
检查方法：打开Dominator Tree，查看大内存对象及其支配对象，结果如下：
{% asset_img 实例二，支配树.png 实例二，支配树 %}
结果分析：okhttp打开缓存会加大不少内存消耗
代码分析：检查代码，okhttp有多个实例使用缓存，某些地方并不需要缓存。
解决方法：不需要缓存的地方将缓存设置去除。

### 六、写在最后
内存问题几乎是每个版本迭代需要关注的问题，可能一小的泄漏点就会导致崩溃，卡顿，ANR等各种问题，建议在版本测试周期或者灰度之前需要做一次内存检查，保证版本质量。MAT基本可以满足内存的检查要求，基本上你能想到的功能它都有提供，更多细节本文章不再赘述，可参考官方文档或网上资料。


### PS:
[官方文档传送门](https://help.eclipse.org/luna/index.jsp?)

### 参考文献
1. [【腾讯Bugly干货分享】Android内存泄漏的简单检查与分析方法](https://segmentfault.com/a/1190000006884310)
2. [10 Tips for using the Eclipse Memory Analyzer](http://www.eclipsesource.com/blogs/2013/01/21/10-tips-for-using-the-eclipse-memory-analyzer/)
3. [Eight Ways Your Android App Can Leak Memory](http://blog.nimbledroid.com/2016/05/23/memory-leaks.html)
