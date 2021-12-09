---
layout: post
title:  "基于API 30 的Activity启动流程分析"
date:   2016-07-30 20:30:00
catalog:  true
tags:
    - android
    - 组件系列
    

---

> 基于Android 6.0源码剖析，本文涉及的相关源码：

# Warning！本文基于API 29，基于 API 30 的Activity启动流程分析已更新，点击：[Activity启动流程？基于Api30的Activity启动流程分析 https://www.jianshu.com/p/b68914037364](https://www.jianshu.com/p/b68914037364)

![img](https://s2.loli.net/2021/12/09/kvLSDoJWmaQezsF.png)

Activity的启动流程api26

直接放图。

*第一次做这个图的时候，我是按照api26的源码来的，然而现在已经api29了……*

*Activity的启动过程的源码我看了多个版本的，每个版本都有一些变化，有些地方属实搞得人摸不着头脑。尤其是中间这几个类，一直在改动。比如这个`ActivityStarter`，如果我没记错的话是api23还是26才加进来的；后来又把`ActivityStackSupervisor`的一部分功能分离到了不同的类中。看看源码中的注释：*

![img](https:////upload-images.jianshu.io/upload_images/6532223-57d9e7ee71d46b4b.png?imageMogr2/auto-orient/strip|imageView2/2/w/903/format/webp)

ActivityStackSupervisor



*好吧，这个类即将被移除了……*

个人感觉可能是Android团队前期埋的坑太多，留下了一些超巨型的类，很多类都超过了几千上万行；然后随着版本的更替，发现这么多功能都放一个类里面也不成，又在慢慢分离一些职责到其他类中，所以就造成了一个版本一个样，变化很快。代码流程随着版本优化，这也是必然的趋势。

不过呢，总体来讲，大体的流程框架基本是固定的，太细节的地方也不需要过多纠结了。

言归正传，从顶至下，**我个人**把Activity的启动流程依次分为**三个阶段**：
 **App进程中** ——(通过Binder)—→ **系统进程中** ——(通过Binder)—→ **回到App进程中**

下面依次进行梳理。（只保留关键代码）

# 1. App进程中

App进程第一轮做的事儿不多，主要就是把传进来的Intent扔给AMS。

## 1.1 Activity

在Activity中调用了`startActivity`方法后，不管调用的是哪个重载，最后都会进入到`startActivityForResult(Intent, int, Bundle)`中。



```kotlin
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    ……
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
    ……
    }
```

然后就进入到了`Instrumentation.execStartActivity`方法中。

## 1.2 Instrumentation



```csharp
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    ……
            ActivityTaskManager.getService().startActivity(
                        whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
    ……
    }
```

**注意！重点！**
 在之前的版本中，`Instrumentation`都是通过Binder的方式调用AMS的方法启动Activity的；但是在api29中，IPC的对象变成了**ActivityTaskManagerService（ATMS）**。根据这个类的说明，“*此类提供有关Activity及其容器（如task，stack和display）的信息，并与之交互。*”网上关于这个类的信息不多，但是通过源码可以发现，这个类实际上就是AMS分离出的一部分职责（毕竟AMS两万多行，也该瘦瘦身了）。
 `Instrumentation.execStartActivity`通过`ActivityTaskManager.getService()`获得了一个`IActivityTaskManager`类型的对象，这是一个Binder对象，负责应用和ATMS之间的通信。也就是在这里，流程进入到了第二阶段：系统进程中。

# 2. 系统进程中

## 2.1 ATMS（ActivityTaskManagerService）

**ATMS**运行在系统服务进程（system_server）之中。当App通过Instrumentation和ATMS跨进程通信之后，ATMS就代管了接下来的启动流程。
 ATMS在进行了简单处理之后，就会交给**`ActivityStarter`**处理。（这其实跟之前的AMS一模一样）



```java
    public final int startActivity(...args) {
        return startActivityAsUser(...args);
    }

    public final int startActivityAsUser(args) {
        // ...
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .......
                .execute();
    }
```

上述代码中，`getActivityStartController().obtainStarter`会返回一个`ActivityStarter`对象。（*这个`ActivityStartController`中有一个`ActivityStarter.DefaultFactory`类型的成员变量，它维护了一个ActivityStarter池。`obtainStarter()`就是从里面取的。*）
 然后`ActivityStarter.execute()`会根据传入的字符串`"startActivityAsUser"`来进行接下来的流程。

## 2.2 一些重要的类

接下来的工作，就是一系列的类协同处理了。代码非常复杂，但是看个大概流程还是不难的，毕竟Android源代码近亿行，不可能全部都仔细读完。

这部分主要涉及了几个类：`ActivityStarter`、`ActivityStackSupervisor`、`ActivityRecord`、`TaskRecord`、`ActivityStack`。

**ActivityStarter**
 顾名思义，`ActivityStarter`是用来启动Activity的，同时也在其中判断了启动模式的逻辑；

**ActivityStackSupervisor**
 顾名思义，`ActivityStackSupervisor`则是ActivityStack的管理者。

**ActivityRecord**
 而每次启动一个Activity，都有一个对应的`ActivityRecord`被记录下来。这个ActivityRecord是在`ActivityStarter.startActivity(IApplicationThread, ...)`方法中被创建的，也就是在这里，Activity的启动信息完成了从intent到ActivityRecord的蜕变。`ActivityRecord`包含了一个Activity的所有信息，可以看做是Activity的“身份证”。

**TaskRecord**
 **[任务](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Factivities%2Ftasks-and-back-stack)**。一个`TaskRecord`就是在四种启动模式中所说的“栈”。一个`TaskRecord`，或者说一个Task、任务，根据官方说法，**任务是用户在执行某项工作时与之互动的一系列 Activity 的集合。**也就是说，一个任务是包含了若干个Activity的。对于启动模式为singleInstance的Activity来说，会单独存在于一个栈中，也就是这个任务中只有一个Activity；而其他启动模式的Activity则会在当前栈中进行。具体的参考官网[定义启动模式](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Factivities%2Ftasks-and-back-stack%23TaskLaunchModes)。

**ActivityStack**
 ActivityStack顾名思义，也就是Activity栈。通常来讲，一个系统里面有两个ActivityStack，一个是系统Launcher所在的ActivityStack，一个是用户App运行的ActivityStack。通常App所在的栈就是后者。
 一个`ActivityStack`包含了多个`TaskRecord`，如图所示：[图片来源](https://www.jianshu.com/p/94816e52cd77)

![img](https://s2.loli.net/2021/12/09/18t5KTC4MXJaUIn.png)

偷图



## 2.3 ActivityStarter

顾名思义，ActivityStarter肩负着Activity启动的职责。之前说到，ATMS调用了`ActivityStarter.execute()`方法，`execute()`又会调用`startActivityMayWait()`方法。

**startActivityMayWait**
 为什么这个方法名字里有个“MayWait”呢？因为接下来的流程都是在同步代码块里面进行的，保证了Activity的启动在多线程下的安全性。这个方法会做一系列的验证和处理，比如根据Intent来从PackageManagerService收集待启动Activity的信息（AndroidManifest中的信息）。

**startActivity**
 `startActivityMayWait`最终会调用`startActivity`。`startActivity`共有三个重载，关键是在这个流程里面，这三个重载挨个都被调用了……不得不吐槽，Android这个方法名字取得真的毫无特色，不仔细看根本不知道他是干啥的；这三步我就合到一起写了。过程很冗长，大多是做一些条件的判断，以及继续收集目标Activity的信息。最重要的是，在这个过程中，根据收集到的这些信息，**目标Activity对应的`ActivityRecord`终于被创建了**。

**startActivityUnchecked**
 经过三个`startActivity`方法之后，终于到了重头戏了。在`startActivityUnchecked`方法中，包含了关于**启动模式launchMode**的逻辑。根据任务栈中是否已经存在该Activity实例、AndroidManifest中设置的启动模式和传入Intent中的flags等，判断出目标Activity的启动方式（[《定义启动模式》](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Factivities%2Ftasks-and-back-stack%23TaskLaunchModes)有详尽的关于启动模式的说明）。
 在判断完Activity的启动方式之后，`startActivityUnchecked`方法中有这么一句代码：



```css
mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition, mOptions);
```

看到这个方法名，我以为流程就走这继续了呢；结果点进去看了半天才发现，这个方法处理的是把目标Activity压入相应的任务栈（或者新建任务栈或者pop上方Activity，诸如此类）相关的逻辑，另外就是创建Starting Window相关的逻辑；这算是个支线任务。真正启动Activity的流程还得往下看。
 后面的逻辑就很清晰了，删去旁支之后：



```cpp
if (mDoResume) {
    // ...
    mRootActivityContainer.resumeFocusedStacksTopActivities(mTargetStack, mStartActivity, mOptions);
} 
```

不过这个mDoResume又是哪儿来的？经过我的不懈努力，终于在某个`startActivity`方法中发现了，其实他就是true：



```dart
final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);
```

OK，到这就很明了了。所以说，到这又跟之前版本不同了，没有经过`ActivityStackSupervisor`，而是调用了`RootActivityContainer.resumeFocusedStacksTopActivities`。

## 2.4 RootActivityContainer / ActivityStack / ActivityStackSupervisor

这个`RootActivityContainer`又是何方神圣呢？看看它的注释：

> Root node for activity containers. TODO: This class is mostly temporary to separate things out of ActivityStackSupervisor.java. The intention is to have this merged with RootWindowContainer.java as part of unifying the hierarchy.

> 意思就是，这是个临时类，分离了一部分`ActivityStackSupervisor`的职责，到最后这个类会被合并到`RootWindowContainer`类中；之前这一步的确应该是交给`ActivityStackSupervisor`的。根据源码中的注释所言，之后`ActivityStackSupervisor`类会被彻底消灭掉，类的职责会被其他类分割。
>  总而言之，这个类暂时就是`ActivityStackSupervisor`一部分的替代品，之后这部分工作就应该是`RootWindowContainer`来做了。

**RootActivityContainer.resumeFocusedStacksTopActivities()及之后的一系列方法**

回到`RootActivityContainer.resumeFocusedStacksTopActivities`方法中；这开始之后的调用链依次是：
 `RootActivityContainer.resumeFocusedStacksTopActivities`
 -> `ActivityStack.resumeTopActivityUncheckedLocked`
 -> `ActivityStack.resumeTopActivityInnerLocked`

它们的共同作用就是确保刚才压入栈的目标Activity能够顺利resume，包括各种状态的判断、暂停当前Activity、处理过渡动画等。而在`ActivityStack.resumeTopActivityInnerLocked`方法的最后：



```ruby
if (next.attachedToProcess()) {
    // ...
} else {
    mStackSupervisor.startSpecificActivityLocked(next, true, true);
}
```

这个`attachedToProcess`方法的返回值也折腾了我很久；最后终于发现，只有启动了的Activity才会返回true。所以代码最终进入到了`ActivityStackSupervisor.startSpecificActivityLocked`方法中。此时第二阶段的流程已经接近尾声。

**`ActivityStackSupervisor.startSpecificActivityLocked()`**中又调用了
 **`ActivityStackSupervisor.realStartActivityLocked()`**
 终于，看到这个名字就知道，这是真正的启动Activity了！

**ActivityStackSupervisor.realStartActivityLocked(ActivityRecord, WindowProcessController, boolean, boolean)**
 这个方法有200多行，但是最关键的其实就是下面这一段：



```java
    final ClientTransaction clientTransaction = ClientTransaction.obtain(
            proc.getThread(), r.appToken);
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
            System.identityHashCode(r), r.info,
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
            r.icicle, r.persistentState, results, newIntents,
            dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                    r.assistToken));
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
```

这里的mService是ATMS，它通过`LifecycleManager`传递了一个Activity启动的事务；`scheduleTransaction`是这样的：



```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        // ...
    }
```

拨云见日！熟悉的`IApplicationThread`，终于结束了在系统进程中的流程，又要回到App进程了。

# 3. 重回App进程

继续偷图，原文https://www.jianshu.com/p/42cc38ba6112

![img](https://s2.loli.net/2021/12/09/8iEHGAmOtCqbIxN.png)

https://upload-images.jianshu.io/upload_images/6635796-9f90882946ad4590.png



**参考资料**
 https://www.jianshu.com/p/42cc38ba6112
 https://www.jianshu.com/p/94816e52cd77
 [https://developer.android.google.cn/guide/components/activities/tasks-and-back-stack](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Factivities%2Ftasks-and-back-stack)



作者：littlefogcat
链接：https://www.jianshu.com/p/160a53701ab6
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。