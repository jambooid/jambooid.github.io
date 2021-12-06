---
layout:     post
title:      android插件化基础-理解context
subtitle:   This is subtitle
date:       2019-04-21
author:     Jambooid
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Tylor
---

Android插件化基础的主要内容包括

> - [Android插件化基础1-----加载SD上APK](https://www.jianshu.com/p/ce71e096ebdf)
> - [Android插件化基础2----理解Context](https://www.jianshu.com/p/e6ce2d03f8f9)
> - [Android插件化基础3----Android的编译打包APK流程详解](https://www.jianshu.com/p/85c8ce13fcad)
> - Android插件化基础4----APK安装流程详解(请期待)
> - Android插件化基础5----Rsources.arsc详解(请期待)
> - Android插件化基础6----Android的资源系统(请期待)
> - Android插件化基础7----Activity的启动流程(请期待)
> - Android插件化基础8----如何启动一个没有注册过的Activity(请期待)
> - Android插件化基础9----Service的启动流程(请期待)
> - Android插件化基础10----BroadcastReceiver源码解析(请期待)

为了让大家在后面更好的理解插件化的内容，我们本篇文章围绕Context(基于Android API 24)进行讲解，主要内容如下：

> - 1、前言
> - 2、Context的概念
> - 3、Context的族谱
> - 4、Context家族成员源码分析
> - 5、初始化过程
> - 6、APP各种Context访问资源的唯一性详解
> - 7、Context的内存泄露

## 一、前言：

Context在android 系统中的地位不言而喻，而且Context对于我们Android开发同学来说，也并不陌生。在我们平时工作中我们经常会用Context来获取APP的资源，开启Activity,获取系统Service服务，发送广播等，经常有人会问：

> Context到底是什么？
> Context如何实现以上功能的？
> 为何不同的Context访问资源都得到的是同一套资源？
> 为何不同场景下要使用不同的Context？
> 一个APP中有多少个Context？
> getApplication和getApplicationContext是同一个对象吗？

那么我们就带着以上的内容来开始我们今天的内容。

## 二、Context的概念

### (一)、源码解读

由于Context的源码内容太多，4K多行代码，太多了，我们就简单的看下类的的注释



```dart
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
public abstract class Context {
   //省略代码
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
   //省略代码
}
```

我们来简单的翻译一下注释：

> 获取一个应用程序全部信息的接口。它是一个抽象类，由Android系统提供它的具体实现类，它拥有访问应用程序内特定的资源和类的权限，以及应用程序级操作，例如启动activity、广播和接收intent等。

上面是官方回答，我的理解是：

> - 从抽象的角度来理解：
>   咱们平时在工作生活中经常会使用到一个词儿——"场景"，一个使用场景就代表用户和我们的app软件交互的过程。比如，你在家里打”农药“就是一个场景；你在宾馆和妹子一起用手机看"小"电影也是一个场景；你在公司用电脑调试你的app也是一个场景。那么Context 就可以抽象的理解为一个场景，每个场景有不同的内容。在程序中，我们可以理解为某对象在程序中所处的"场景"(环境)，一个与系统交互的过程，比如上面你在打”农药“，你的场景就是你的操作界面，和你的手势相关操作的数据与传输。所以Context在加载资源、启动Activity、获取系统服务、创建View等操作都要参与进来。打电话、发短信、玩"农药"等都是有界面的，所以是一类"有界面的"场景，还有一些没有界面的场景，比如你在后台播放歌曲的程序，就是一类"没有界面"的场景。其实一个app就描绘了一个主要的场景，比如淘宝，描绘的是购物的场景；微信是聊天的场景；支付宝，描绘的就是支付的场景，所以说Context其实就是一个"场景"，因为Context是一个"场景"，所以它可以获取这个场景下的所有信息。
> - 从研发的角度来理解：
>   Context是一个抽象类，我们通过这个Context可以访问包内的资源(res和assets)和启动其他组件(activity、service、broadcast)以及系统服务(systemService)等。所以Context提供了一个应用程序运行环境，在Context的环境里，应用才可以访问资源，才能和其他组件、服务交互，Context定义了一套基本功能接口，我们可以理解为一套规范，而Activity和Service是实现这套规范的具体实现类(其实内部是ContextImpl统一实现的)。所以可以这样说，Context是维持Android程序中各个组件能够正常工作的一个核心功能类。

上面说了Context是一个抽象类，那它的具体子类都有哪些？我们来一起看一下他的族谱。

## 三、Context的族谱

那我们来看下他们的具体子类，在[Context官网的API](https://link.jianshu.com?t=https://developer.android.com/reference/android/content/Context.html)如下:

![img](D:\github\blog\jambooid.github.io\_posts\typora-img\context 族谱)

官网的API.png


 图片太大，加上很多类咱们暂时不需要考虑，整理了下，如下图:



![img](D:\github\blog\jambooid.github.io\_posts\typora-img\context类图)

Context类及其子类的继承关系.png

我们以activity为本体，以继承的形式来分析，可以大概分为4辈儿，其中activity是本级；ContextThemeWrapper是Activity的父类，Service、Aplication是Actiivty的叔类；ContextWrapper是Activity的祖父类、ContextImpl是Activity的叔祖父类；Context是Activity的曾祖父类。

我们再自上而下的去讲解下：

> - 1、Context类是一个纯粹的抽象类
> - 2.1、ContextImpl是Context的具体实现类，实现了Context的所有功能。
> - 2.2、ContextWrapper，通过字面就知道是一个Context的包装类，由于ContextWrapper就一个构造函数，且必须要传入一个Context，所以ContextWrap中持有有一个mBase变量的应用，通过调用attachBaseContext(Context)方法用于给ContextWrapper对象中mBase指定真正的Context对象。传入的变量就是ContextImpl对象。
> - 3.1、ContextThemeWrapper内部封装了包含了与主题相关的接口，这样就可以在AndroidManifest.xml文件中添加android：theme为Application或者Activity指定的主题，所以Actvity继承ContextThemeWrapper
> - 3.2、由于Application不需要界面，所以直接继承自ContextWrapper。
> - 3.3、由于Service同样不需要界面，所以也直接继承自ContextWrapper。
> - 4、由于Activity需要主题，所以Activity继承自ContextThemeWrapper。

PS:因此在绝对大多数场景下，Activity、Service和Application这三种类型的Context都是可以通用的，因为他们都是ContextWrapper的子类，但是由于如果是启动Activity，弹出Dialog，一般涉及到"界面"的领域，都只能由Activity来启动。因为出于安全的考虑，Android是不允许Activity或者Dialog凭空出现的，一个Activity的启动必须建立在另一个Activity的基础之上，也就是以此形成了返回栈。而Dialog则必须在一个Activity上面弹出(如果是系统的除外)，因此这种情况下我们只能使用Activity类型的Context。整理了一些使用场景的规则，也就是Context的作用域，如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-e3f22664fd48e65d.png?imageMogr2/auto-orient/strip|imageView2/2/w/589/format/webp)

Context作用域.png

从上图，我们知道Activity作为Context，作用域最广，是因为Activity继承自ContextThemeWrapper，而Application和Service继承自ContextWrapper。很显然ContextThemeWrapper在ContextWrapper的基础上又做了一些操作使Activity异常强大，关于源码，我们下面会讲解。YES和NO，太明显就不提了，简单说下不推荐的几种情况：

> - 1、如果我们用Application作为Context去启动一个没有设置过LaunchMode的Activity会报错"android.util.AndroidRuntimeException: Calling startActivity from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?"，这是因为非Activity类型的Context并没有所谓的任务栈，所以待启动的Activity就找不到栈了。解决这个问题方法就是为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK的标志位，这样启动这个Activity，它就会创建一个新的任务栈，此时Activity是以singleTask模式启动的，所有这种用Application启动Activity方式不推荐使用，Service同理。
> - 2、在Application和Service中使用LayoutInflater也是合法的，但是会使用系统默认的主题样式，这样你自定义的样子可能会受到影响，所以这种方式也不推荐使用。

总结一下：
 和"界面"(UI)有关的，都应该使用Activity作为Context来处理，其他的可以使用Service、Application或者Activity作为Context。关于Context内存泄露的问题，下面再详细将讲解。

## 四、源码分析初始化过程

#### (一)、[ContextWrapper](https://link.jianshu.com?t=https://developer.android.com/reference/android/content/ContextWrapper.html)源码解析

由于ContextWrapper源码比较多，我只截取了一部分，源码如下：



```java
/**
 * Proxying implementation of Context that simply delegates all of its calls to
 * another Context.  Can be subclassed to modify behavior without changing
 * the original Context.
 */
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }

    /**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }

    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }

    @Override
    public Resources getResources() {
        return mBase.getResources();
    }

    @Override
    public PackageManager getPackageManager() {
        return mBase.getPackageManager();
    }

    @Override
    public ContentResolver getContentResolver() {
        return mBase.getContentResolver();
    }
  //省略了部分代码 
}
```

ContextWrapper继承自Context，和我们之前的画的类继承图一样。

##### 1、类注释翻译

> Context的代理类，具体的调用由另外一个Context来实现，除了不能修改真正的Context，其他方面子类是可以修改的。

> - 注释说的很明显ContextWrapper是个代理类，之前我们已经讲解过了，这是明显的静态代理，起到真正作用的是Context mBase。
> - 大家发现ContextWrapper的所有方法均是调用的mBase对应的方法，所以进一步证明了ContextWrapper是代理，mBase才是真正的执行者。
> - 所以说它的三个子类ContextThemeWrapper，Service，Application也是不能改变具体由mBase的地址
> - 该类的构造函数包含了一个真正的Context引用（ContextImpl对象），然后就变成了ContextImpl的装饰着模式。

##### 2、attachBaseContext(Context)方法解析

方法注释翻译：

> 通过这个方法来设置ContextWrapper的字段mBase赋值，所有的调用其实是代理调用到mBase这个Context，如果字段mBase已经被赋值了，则会抛出异常。

这个方法其实很简答， 就是判断是否被赋值过，赋值过则抛异常，没有给mBase赋值

#### (二)、[ContextThemeWrapper](https://link.jianshu.com?t=https://developer.android.com/reference/android/view/ContextThemeWrapper.html)源码解析



```dart
/**
 * A context wrapper that allows you to modify or replace the theme of the
 * wrapped context.
 */
public class ContextThemeWrapper extends ContextWrapper {
     //省略部分代码
    /**
     * Creates a new context wrapper with no theme and no base context.
     * <p>
     * <stong>Note:</strong> A base context <strong>must</strong> be attached
     * using {@link #attachBaseContext(Context)} before calling any other
     * method on the newly constructed context wrapper.
     */
    public ContextThemeWrapper() {
        super(null);
    }

    /**
     * Creates a new context wrapper with the specified theme.
     * <p>
     * The specified theme will be applied on top of the base context's theme.
     * Any attributes not explicitly defined in the theme identified by
     * <var>themeResId</var> will retain their original values.
     *
     * @param base the base context
     * @param themeResId the resource ID of the theme to be applied on top of
     *                   the base context's theme
     */
    public ContextThemeWrapper(Context base, @StyleRes int themeResId) {
        super(base);
        mThemeResource = themeResId;
    }

    /**
     * Creates a new context wrapper with the specified theme.
     * <p>
     * Unlike {@link #ContextThemeWrapper(Context, int)}, the theme passed to
     * this constructor will completely replace the base context's theme.
     *
     * @param base the base context
     * @param theme the theme against which resources should be inflated
     */
    public ContextThemeWrapper(Context base, Resources.Theme theme) {
        super(base);
        mTheme = theme;
    }

    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
    }
   //省略部分代码
}
```

ContextThemeWrapper继承自ContextWrapper，和我们之前的画的类继承图一样

##### 1、类注释

来看下类的注释，翻译如下：

> 一个允许你修改或者替换主题的Context的包装类。

##### 2、构造函数解析

> - 有三个构造函数
> - 无参的构造函数调用的是super(null)，所以说调用无参的构造函数的mBase属性是没有值的，同时主题也是null。
> - 2个参数的构造函数，第二个参数是int类型的构造函数，第一个参数Context给mBase字段赋值，第二个参数给mThemeResource赋值
> - 2个参数的构造函数，第二个参数是Theme类型的构造函数，一个参数Context给mBase字段赋值，第二个参数给mTheme赋值。
> - 通过两个参数的构造函数我们知道，只要不是调用无参的ContextThemeWrapper构造函数，ContextThemeWrapper对象的mBase是一定有值的。

PS：mTheme和mThemeResource是有关联关系的。

##### 3、attachBaseContext(Context newBase)

> 这个方法很简单，就是调用父类的attachBaseContext方法。

#### (三)、[ Service](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/Service.html)源码解析



```java
public abstract class Service extends ContextWrapper implements ComponentCallbacks2 {
    //省略部分代码
  
    public Service() {
        super(null);
    }

    /** Return the application that owns this service. */
    public final Application getApplication() {
        return mApplication;
    }

    public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {
        attachBaseContext(context);
        mThread = thread;           // NOTE:  unused - remove?
        mClassName = className;
        mToken = token;
        mApplication = application;
        mActivityManager = (IActivityManager)activityManager;
        mStartCompatibility = getApplicationInfo().targetSdkVersion
                < Build.VERSION_CODES.ECLAIR;
    }
}
```

关于Service大家都很熟悉了，我就不说类的注释，不过我建议大家还是去看下，对大家进一步理解Service是很有帮助的。Service继承自ContextWrapper，和我们之前的画的类继承图一样

##### 1、构造函数分析

> - 通过构造函数我们知道，Service的构造函数就一个
> - 构造函数默认调用父类的，而且传入的值是null，说明构造Service的时候，mBase是null，那Service的mBase的属性是什么时候被赋值的那？其实他的赋值是在attach(Context,ActivityThread, String, IBinder,Application, Object)方法里面通过调用attachBaseContext(context)赋值的。所以我们可以预测Android系统new (也有可能是反射哦)了Service之后 肯定调用了attach()方法

#### (四)、[Application](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/Application.html)源码解析



```java
public class Application extends ContextWrapper implements ComponentCallbacks2 {
   //省略部分代码
    public Application() {
        super(null);
    }

    final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
   //省略部分代码
}
```

Application继承自ContextWrapper，和我们之前的画的类继承图一样。
 关于Application大家都很熟悉了，我就不说类的注释，不过我建议大家还是去看下，对大家进一步理解Application是很有帮助的。

##### 1、构造函数分析

> - 通过构造函数我们知道，Application的构造函数就一个
> - 构造函数默认调用父类的，而且传入的值是null，说明构造Application实例的时候，mBase是null，那Application的mBase的属性是什么时候被赋值的那？其实他的赋值是在attach(Context)方法里面通过调用attachBaseContext(context)赋值的。所以我们可以预测Android系统new(也有可能是反射哦)了Application之后 肯定调用了attach()方法

#### (五)、[Activity](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/Activity.html)源码解析



```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback {
       //省略部分代码
}
```

Activity继承自ContextThemeWrapper，和我们之前的画的类继承图一样

> 咦！突然发现没有找到Activity的构造函数，哦~，原来Activity.java是没有写构造函数的，所以Activity的是用无参的构造函数，默认会调用父类也就是ContextThemeWrapper的无参构造函数，所以Activity对象实例化的时候mBase是没有值，那Activity的mBase是什么时候被赋值的那？我们找一下发现是在调用***attach(Context, ActivityThread, Instrumentation,IBinder, int,Application, Intent, ActivityInfo,
> CharSequence, Activity, String,NonConfigurationInstances,
> Configuration,String,IVoiceInteractor,Window)attachBaseContext(context)\*** 方法里面调用了attachBaseContext(context);给Activity的mBase赋值的。

#### (六)、[ContextImpl](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/Activity.html)源码解析

ContextImpl继承自Context ，和我们之前的画的类继承图一样



```java
/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {
    final LoadedApk mPackageInfo;
    //省略部分代码
    private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, int flags,
            Display display, Configuration overrideConfiguration, int createDisplayWithId) {
        mOuterContext = this;

        // If creator didn't specify which storage to use, use the default
        // location for application.
        if ((flags & (Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE
                | Context.CONTEXT_DEVICE_PROTECTED_STORAGE)) == 0) {
            final File dataDir = packageInfo.getDataDirFile();
            if (Objects.equals(dataDir, packageInfo.getCredentialProtectedDataDirFile())) {
                flags |= Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE;
            } else if (Objects.equals(dataDir, packageInfo.getDeviceProtectedDataDirFile())) {
                flags |= Context.CONTEXT_DEVICE_PROTECTED_STORAGE;
            }
        }

        mMainThread = mainThread;
        mActivityToken = activityToken;
        mFlags = flags;

        if (user == null) {
            user = Process.myUserHandle();
        }
        mUser = user;

        mPackageInfo = packageInfo;
        mResourcesManager = ResourcesManager.getInstance();

        final int displayId = (createDisplayWithId != Display.INVALID_DISPLAY)
                ? createDisplayWithId
                : (display != null) ? display.getDisplayId() : Display.DEFAULT_DISPLAY;

        CompatibilityInfo compatInfo = null;
        if (container != null) {
            compatInfo = container.getDisplayAdjustments(displayId).getCompatibilityInfo();
        }
        if (compatInfo == null) {
            compatInfo = (displayId == Display.DEFAULT_DISPLAY)
                    ? packageInfo.getCompatibilityInfo()
                    : CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO;
        }

        Resources resources = packageInfo.getResources(mainThread);
        if (resources != null) {
            if (displayId != Display.DEFAULT_DISPLAY
                    || overrideConfiguration != null
                    || (compatInfo != null && compatInfo.applicationScale
                            != resources.getCompatibilityInfo().applicationScale)) {

                if (container != null) {
                    // This is a nested Context, so it can't be a base Activity context.
                    // Just create a regular Resources object associated with the Activity.
                    resources = mResourcesManager.getResources(
                            activityToken,
                            packageInfo.getResDir(),
                            packageInfo.getSplitResDirs(),
                            packageInfo.getOverlayDirs(),
                            packageInfo.getApplicationInfo().sharedLibraryFiles,
                            displayId,
                            overrideConfiguration,
                            compatInfo,
                            packageInfo.getClassLoader());
                } else {
                    // This is not a nested Context, so it must be the root Activity context.
                    // All other nested Contexts will inherit the configuration set here.
                    resources = mResourcesManager.createBaseActivityResources(
                            activityToken,
                            packageInfo.getResDir(),
                            packageInfo.getSplitResDirs(),
                            packageInfo.getOverlayDirs(),
                            packageInfo.getApplicationInfo().sharedLibraryFiles,
                            displayId,
                            overrideConfiguration,
                            compatInfo,
                            packageInfo.getClassLoader());
                }
            }
        }
        mResources = resources;

        mDisplay = (createDisplayWithId == Display.INVALID_DISPLAY) ? display
                : mResourcesManager.getAdjustedDisplay(displayId, mResources.getDisplayAdjustments());

        if (container != null) {
            mBasePackageName = container.mBasePackageName;
            mOpPackageName = container.mOpPackageName;
        } else {
            mBasePackageName = packageInfo.mPackageName;
            ApplicationInfo ainfo = packageInfo.getApplicationInfo();
            if (ainfo.uid == Process.SYSTEM_UID && ainfo.uid != Process.myUid()) {
                // Special case: system components allow themselves to be loaded in to other
                // processes.  For purposes of app ops, we must then consider the context as
                // belonging to the package of this process, not the system itself, otherwise
                // the package+uid verifications in app ops will fail.
                mOpPackageName = ActivityThread.currentPackageName();
            } else {
                mOpPackageName = mBasePackageName;
            }
        }
        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }

    static ContextImpl createSystemContext(ActivityThread mainThread) {
        LoadedApk packageInfo = new LoadedApk(mainThread);
        ContextImpl context = new ContextImpl(null, mainThread,
                packageInfo, null, null, 0, null, null, Display.INVALID_DISPLAY);
        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                context.mResourcesManager.getDisplayMetrics());
        return context;
    }

    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        return new ContextImpl(null, mainThread,
                packageInfo, null, null, 0, null, null, Display.INVALID_DISPLAY);
    }

    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        return new ContextImpl(null, mainThread, packageInfo, activityToken, null, 0,
                null, overrideConfiguration, displayId);
    }
   //省略部分代码
}
```

如果大家第一次找ContextImpl类，还是比较麻烦的，因为ContextImpl类是ReceiverRestrictedContext的内部类。这个类的代码比较多，这里就不详细说明了，大家可以自己去了解下
 通过类的注释我们得知：

> Context API的通用实现，它为Activity和其他应用程序组件提供了基础的Context。

> - 上面我们发现ContextImpl的构造函数是private，所以外部想要获取ContextImpl的实例，所以通过它的其他方法
> - createSystemContext(//省略入参)方法可以获取系统的ContextImpl
> - createAppContext(//省略入参)方法获取获取Application的ContextImpl
> - createActivityContext(省略入参)方法获取Activity的ContextImpl

## 五、Context家族成员初始化分析

#### (一)、ContextImpl 实例在哪里创建

每一个应用程序在客户端都是从ActivityThread类(重点提醒:ActivityThread不是Thread)开始的，创建Context对象也是在该类中完成，具体创建ContextImpl的地方一种有6处

> - LoadedApk.makeApplication(); //772 line
> - ActivityThread.createBaseContextForActivity();//2674  line
> - ActivityThread.handleCreateBackupAgent  //3061  line
> - ActivityThread.handleCreateService  //3163  line
> - ActivityThread.attach();  // 5930  line
> - ActivityThread.getSystemContext()  //2052 line

也有人说在PackageInfo类的makeApplication()也有，但是我没找到，如果有朋友找到也留言告诉我一声，谢谢！

通过上面方法我们发现貌似大部分ContextImpl的实例创建都是在ActivityThread里面，可见ActivityThread很重要。

#### (二)、Application及对应的mBase实例创建过程

源码说话:



```kotlin
    //ActivityThread.java
    private void handleBindApplication(AppBindData data) {
          //省略部分代码
        
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers);
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
      //省略部分代码
```

在***ActivityThread的handleBindApplication(AppBindData)\***里面调用 ***data.info.makeApplication(boolean,Instrumentation);\***，因为 ***data\***是 ***AppBindData\*** 类的实例， ***data.info\*** 是 ***LoadedApk\***对象，所以data.info.makeApplication(boolean,Instrumentation)其实调用的是 ***LoadedApk\***的
 ***makeApplication((boolean,Instrumentation)\***方法，那我们进去看下



```csharp
 public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");
        Application app = null;
        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }
        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }

************************重点******************************
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
************************重点******************************
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
       //省略部分代码
    }
```

看我画重点的部分

> - 1、里面调用***ContextImpl.createAppContext(ActivityThread, LoadedApk)\***方法获取一个 ***ContextImpl\*** 对象
> - 2、调用 ***Instrumentation的newApplication(ClassLoader ,String, Context)\*** 方法创建一个 ***  Application*** 。
> - 3、***appContext.setOuterContext(app)\*** 来设置 ***Application的ContextImpl\***

至此完成了 ***Application\*** 及其 ***mBase\*** 的实例创建过程，那我们再来看下 ***Instrumentationd的newApplication(ClassLoader ,String, Context)\*** 方法的源码



```java
  public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        return newApplication(cl.loadClass(className), context);
    }

    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);
        return app;
    }
```

> 我们发现 ***Instrumentationd的newApplication(ClassLoader ,String, Context)\*** 方法内部又调用 ***Instrumentationd\*** 的静态方法 ***newApplication(Class<?> clazz, Context context)\*** 来实现的，在 ***Instrumentationd\*** 的静态方法 ***newApplication(Class<?> clazz, Context context)\*** 里面是调用反射的newInstance()方法来获取的。原来 ***Application\*** 是通过反射来实例化的啊。

#### (三)、Activity及对应的mBase实例创建过程

> - 启动Activity时，AMS(ActivityManagerService)会通过IPC调用到
>   ***ActivtyThread的scheduleLaunchActivity()\*** 方法，该方法包含两个参数(不是只有两个参数)。一种是 ***ActivityInfo\*** ，这是一个实现了 ***Parcelable\*** 接口的数据类，意味着该对象是AMS创建的，并通过IPC传递到 ***ActivityThread\*** ;另一种是其他的一些参数。



```java
             @Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
            updateProcessState(procState, false);
            ActivityClientRecord r = new ActivityClientRecord();
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;
            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;
            r.startsNotResumed = notResumed;
            r.isForward = isForward;
            r.profilerInfo = profilerInfo;
            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

H类其实是一个Handler



```java
  private class H extends Handler {
        //省略部分代码
       public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
            //省略部分代码
         }
      //省略部分代码
}
```

> ***scheduleLaunchActivity()\*** 方法中会根据以上两种参数构造一个 ***ActivityRecord\*** 数据类， ***ActivityThread\*** 内部会为每一个 ***Activity\*** 创建一个 ***ActivityRecord\*** 对象，并使用这些数据对象来管理 ***Activity\*** 。 ***scheduleLaunchActivity()\*** 方法执行完后会调用发送一个Message到H，H类重写了一个 ***handleMessage()\*** ,然后会调用到 ***case LAUNCH_ACTIVITY中\*** ，在里面又调用了 ***handleLaunchActivity(ActivityClientRecord , Intent, String)\***  ，那我们进去 ***handleLaunchActivity\*** 方法内部看个究竟



```csharp
     //ActiviyThread.java
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
         //省略部分代码
        Activity a = performLaunchActivity(r, customIntent);
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

            if (!r.activity.mFinished && r.startsNotResumed) {
          
                performPauseActivityIfNeeded(r, reason);

                if (r.isPreHoneycomb()) {
                    r.state = oldState;
                }
            }
        } else {
             //省略部分代码
        }
    }
```

在 ***handleLaunchActivity(ActivityClientRecord , Intent, String)\*** 方法调用 ***performLaunchActivity()\*** 来获取一个Activity，我们猜到其实是 ***performLaunchActivity(ActivityClientRecord , Intent)\*** 方法内部才是真正创建Activity的地方，那我们就来看下源码，代码如下：



```csharp
     //ActiviyThread.java
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //省略部分代码
************************重点******************************
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
            //省略部分代码
            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                //省略部分代码
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
************************重点******************************
              //省略部分代码
    }
```

大家看重点区域,通过阅读源码，我们可以获取如下信息：

> - 1、 ***activity\*** 创建时通过调用 ***Instrumentationd的newActivity(ClassLoader, String,Intent)\*** 方法来获取的
> - 2、里面调用 ***ContextImpl的(ActivityClientRecord , Activity)\*** 方法获取一个 ***ContextImpl\*** 对象
> - 3、 ***Activity 的attach(Context,ActivityThread,Instrumentation, IBinder, int,Application, Intent, ActivityInfo,CharSequence, Activity, String,NonConfigurationInstances,Configuration, String, IVoiceInteractor,Window)\*** ;来设置 ***Activity的ContextImpl\***

那我们来看下 ***Instrumentation的newActivity(ClassLoader, String,Intent)\*** 方法里面是如何实现的



```java
   public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```

其实很简单，就是直接调用放射的newInstance()来获取一个对象。

至此，Activity及对应的mBase实例创建过程已经完成

#### (四)、Service及对应的mBase实例创建过程

> 启动 ***Service\*** 时， ***AMS\*** 会通过调用IPC调用到 ***ActivityThread的scheduleCreateService()\*** 方法，该方法也包含两种参数，第一个是 ***ServiceInfo\*** ，这是一个实现了 ***Parcelable接口\*** 的数据类，该对象由Ams创建，并通过IPC传递到 ***ActivityThread\*** 内部；第二种是其他参数。



```java
         //ActivityThread.java
        public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            sendMessage(H.CREATE_SERVICE, s);
        }
```

H类其实是一个Handler



```java
  private class H extends Handler {
        //省略部分代码
       public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                    case CREATE_SERVICE:
                    //省略部分代码
                    handleCreateService((CreateServiceData)msg.obj);
                    //省略部分代码
                    break;
            //省略部分代码
         }
      //省略部分代码
}
```

> - 在 ***scheduleCreateService()\*** 方法中，会使用以上两种参数构造一个 ***CreateServiceData\*** 的数据对象， ***ActivityThread\*** 会为其所包含的每一个 ***Service\*** 创建该数据对象，并通过这些对象来管理 ***Service\*** 。
> - ***scheduleCreateService()\*** 方法执行完后会调用发送一个Message到H， ***case CREATE_SERVICE中\*** ，在里面又调用了 ***handleCreateService(CreateServiceData)\*** ，那我们进去 ***handleCreateService\*** 方法内部看个究竟



```kotlin
private void handleCreateService(CreateServiceData data) {
************************重点******************************
        //省略部分代码
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
            //省略部分代码
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
************************重点******************************
           //省略部分代码
    }
```

通过阅读源码我们知道

> - 1、 ***Application\*** 和 ***Activity\*** 的创建都是通过 ***Instrumentation\*** 来闯将实例的，不过 ***Service\*** 不是，直接在 ***ActivityThread\*** 里面直接调用反射获取实例
> - 2、里面调用 ***ContextImpl.createAppContext(ActivityThread, LoadedApk)\*** 方法获取一个 ***ContextImpl\*** 对象，这里和 ***Application一样\*** ，和 ***Activity\*** 不一样。
> - 3、调用 ***Service的attach(context,ActivityThread, String, IBinder,Application, Object)\*** 方法进行绑定 ***ContextImpl\*** 。

至此完成了Service及其mBase的实例创建过程

#### (五)、这几个Context之间的关系

> - 从以上可以看出，创建Context对象的过程基本一样，不同的仅仅是针对Application、Activity、Service使用了不同的数据对象。
> - 一个应用程序的Context的个数应该为：Context个数=Service个数+Activity个数+Application个数，如果是普通应用Application个数是一个，如果是插件化中的多进程应用，则Application是多进程个数。
> - 应用程序中包含多个ContextImpl对象，而内部变量mPackageInfo却指向了同一个LoadedApk对象，这种设计结构意味着ContextImpl中大多数进行包操作的重量级函数实际上都转向了mPackageInfo对象的响应方法，也就是事实上调用了同一个LoadedApk对象。

#### (六)如何获取Context

通常我们想要获取Context对象，主要有以下四种方法

> - 1、View.getContext(),返回的是当前View对象的Context对象，通常是当前正在展示的Activity对象
> - 2、context.getApplicationContext()，获取当前Context所在的应用进程的Context对象，通常我们使用Context对象时，要优先考虑这个全局的进程Context。
> - 3、ContextWrapper.getBaseContext();用来获取一个ContextWrapper进行装饰之前的Context，可以使用这个方法，这个方法在实际开发中使用并不多，也不建议使用。
> - 4、Activity.this，返回当前的Activity实例，如果是UI控件需要使用Activity作为Context对象，但是默认的Toast实际上使用ApplicationContext也可以。

#### (七)getApplication()和getApplicationContext()

上面说到了获取当前Application对象用getAppcationContext，不知道你有没有联想到getAppliation(),这两个方法有什么区别？下面我们就仔细研究下。
 以activiy为例，Activity.getAppliation()返回的是mApplication，代码如下：



```java
    public final Application getApplication() {
        return mApplication;
    }
```

那么mApplication是什么时候被初始化的那，其实如果你刚刚仔细读过源码就是会知道是在activity的attach里面被初始化的



```dart
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
      //省略部分代码
      mApplication = application;
      //省略部分代码
}
```

那调用attach方法中的第6个参数application,是怎么来的？
 那我们回到ActivityThread的performLaunchActivity()方法里面一看究竟。



```cpp
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
          //省略部分代码
           Application app = r.packageInfo.makeApplication(false, mInstrumentation);
         //省略部分代码
           activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
        //省略部分代码
}
```

通过上述代码我们知道app来自LoadedApk的makeApplication()方法里面。
 那我们来看下LoadedApk.makeApplication()里面具体的执行



```csharp
    //LoadedApk.java
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
        //省略部分代码
       Application app = null;
        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }
        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app; 
        //省略部分代码
    }
```

通过上述代码，我们知道在LoadedApk的makeApplication里面首先会判断mApplication是否为空，如果不为空直接返回，如果为空则通过反射实例化一个。那我们来看下getAppliationContext()里面的具体实现，首先说明下getApplicationContext()这个方法是ContextWrapper的方法，代码如下：



```java
   //ContextWrapper.java
    @Override
    public Context getApplicationContext() {
        return mBase.getApplicationContext();
    }
```

所以我们知道,其实调用的是mBase.getApplicationContext()方法。因为我们知道mBase其实就是ContextImpl所以我们看下ContextImpl里面的具体实现



```java
    //ReceiverRestrictedContext.java
    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
```

咦，mPackageInfo不是LoadedApk对象吗？那调用 mPackageInfo.getApplication()的源码是：



```cpp
    Application getApplication() {
        return mApplication;
    }
```

哈哈，大家其实看懂了吧，其实无论是getApplication()还是getApplicationContext()最后其实都是调用的LoadedApk的mApplication。而在一个安装包下LoadedApk是同一个对象，所以getApplication()==getApplicationContext()是true。不知道大家看懂了没。

其实还有一种方法，大家在直接打印getApplication()和getApplicationContext()，最后大家会发现是指向同一个内存地址，强烈建议大家去试试！

那么问题来了，既然这两个方法得到结果都是相同的，那么Android为什么要提供这两个功能重复的方法那？实际上这两个方法在作用域上有比较大的区别。getApplication()方法在语义性上非常强，一看就知道来获取Application的，但是她只在Activity和Service上才能调用。而在一些其他场景，比如BroadcastReceiver中也想获取Application实例，这时就可以借助getApplicationContext()方法如下：



```java
publicclassMyReceiverextendsBroadcastReceiver{

     @Override
     public void onReceive(Context context,Intent intent){
                       ApplicationmyApp=(Application)context.getApplicationContext();
     }
}
```

现在希望大家可以清楚的理解getApplicationContext()与getApplication()。

## 六、APP各种Context访问资源的唯一性详解

之前提过一个问题

> 为何不同的Context访问资源都得到的是同一套资源？

首先，我们来看下Activity源码



```csharp
public abstract class Context {    
    public abstract AssetManager getAssets();    
    public abstract Resources getResources();
}
```

所以我们知道Context的getAssets()和getResources()方法是个抽象类，而我们又知道ContextImpl才是Context的具体实现类，那我们再来看下ContextImpl的源码：



```java
class ContextImpl extends Context {
     //省略部分代码
    private final @NonNull ResourcesManager mResourcesManager;
    private final @NonNull Resources mResources;
    @Override
    public AssetManager getAssets() {
        return getResources().getAssets();
    }
    @Override
    public Resources getResources() {
        return mResources;
    }
    //省略部分代码
}
```

所以我们知道，平时写的各种Context的getAssets()方法和getResources()方法获取的Resources对象就是上面的ContextImpl的成员变量mResources。
 那我们来看下mResources什么时候被初始化的，我们发现其实mResources是在ContextImpl里面被初始化的，代码如下：



```csharp
    private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, int flags,
            Display display, Configuration overrideConfiguration, int createDisplayWithId) {
         //省略部分代码
        mResourcesManager = ResourcesManager.getInstance();
        //省略部分代码
        Resources resources = packageInfo.getResources(mainThread);
        if (resources != null) {
            if (displayId != Display.DEFAULT_DISPLAY
                    || overrideConfiguration != null
                    || (compatInfo != null && compatInfo.applicationScale
                            != resources.getCompatibilityInfo().applicationScale)) {

                if (container != null) {
                    // This is a nested Context, so it can't be a base Activity context.
                    // Just create a regular Resources object associated with the Activity.
                    resources = mResourcesManager.getResources(
                            activityToken,
                            packageInfo.getResDir(),
                            packageInfo.getSplitResDirs(),
                            packageInfo.getOverlayDirs(),
                            packageInfo.getApplicationInfo().sharedLibraryFiles,
                            displayId,
                            overrideConfiguration,
                            compatInfo,
                            packageInfo.getClassLoader());
                } else {
                    // This is not a nested Context, so it must be the root Activity context.
                    // All other nested Contexts will inherit the configuration set here.
                    resources = mResourcesManager.createBaseActivityResources(
                            activityToken,
                            packageInfo.getResDir(),
                            packageInfo.getSplitResDirs(),
                            packageInfo.getOverlayDirs(),
                            packageInfo.getApplicationInfo().sharedLibraryFiles,
                            displayId,
                            overrideConfiguration,
                            compatInfo,
                            packageInfo.getClassLoader());
                }
            }
        }
        mResources = resources;
        //省略部分代码
    }
```

这里补充下内容

> - 1、ResourcesManager.getInstance()明显是一个单例
> - 2、对一个App而言，一个App对应一个LoadedApk，所以packageInfo.getResources(mainThread);也应该是同一个Resources
> - 3、由于之前咱们创建的Activity、Service还是Application中的ContextImpl都是中的container=null(至于为什么等于null因为调用的静态create方法里面第一个参数都是null)，所以mResourcesManager也是一份。

综上所述，我们可以看出在设备其他因素不变的情况下我们通过不同的Context实例得到的Resources是同一套资源。

关于Resources我后续会专门开个文章去详细讲解。

## 七、Context的内存泄露问题

因为Activity是生命周期是由系统控制的，所以如果长周期的对象持有对Activit的引用，会导致Activity无法回收。

#### (一)、静态资源导致内存泄露

##### 1、问题

> 由于静态资源在类加载以后就会一直有，声明周期很长，如果它持有一个Activity，会导致Activity无法被回收，而产生内存泄漏。

##### 2、解决方案

> - 1、尽量避免使用静态变量引用Activity
> - 2、如果一定要使用静态变量，则在Activity的onDestroy的时候解除引用，或者指向null。

#### (二)、单例模式导致内存泄露

##### 1、问题

> 单例，意味着这个对象会一直存在，如果这个对象中持有了Activity的引用，也会导致Activity无法被回收，而产生内存泄露

##### 2、解决方案

> 避免在单例模式下持有Context，如果一定要持有也是Application，绝对不能是Activity。

#### (三)、内部类或者匿名内部类导致内存泄露

##### 1、问题

> 内部类(匿名内部类其实也是内部类的一种)会持有外部类的引用，当内部类进行延时操作的时候，如果外部类是Activity，则在执行ondestroy后，并不会销毁，从而导致内存泄露

什么是"内部类持有外部类引用？"
 可以这样理解



```cpp
public class InnerClass {
    private OuterClass outer;
    public InnerClass(OuterClass outer) {
        this.outer = outer;
    }
}
```

就是说，创建InnerClass对象时，必须传递进去一个OuterClass的对象，赋值给InnerClass的一个字段，该字段是OuterClass对象的引用。大家想一下GC的原理，如果InnerClass对象没有被标记为垃圾对象，那么outer指向的OutClass对象没有被标记为垃圾对象，这样就会导致Outer也不会是垃圾对象(GCRoot对InnerClass有引用，而InnerClass对OuterClass又由引用)

##### 2、解决方案

> - 将内部类改为：静态内部类+弱引用
>   因为仅仅改为静态内部类，我们会发现不能调用外部类的方法，所以我们加上弱引用，这样就可以调用外部类的方法了。
> - 这里重点说下AsyncTask
>   如果使用AsyncTask静态内部类，需要保证AsyncTask的周期和Activity的周期保持一致，也就是在Activity的生命周期结束时将要将AsyncTask cancel掉，我个人已经4年不用AsyncTask。
> - 这里重点说下Thread，如果是Thread，也不推荐将Thread生命成static，因为Thread位于GC根部，Delvik VM会和所有活动线程保持hard references关系，所以运行中的Thread绝不会被GC无端回收了，所以正确的解决方法是在自定义静态内部类的基础上给线程加上取消机制，因为我们可以在Activity的onDestory方法中将Thread关闭掉。
> - Timer和TimerTask没有进行Cancel，从而导致Timer和TimerTask会一直持有外部Activity的引用，所以要在适当的时机cancel。

#### (四)、Handler使用导致内存泄露

##### 1、问题

经常会遇见Android程序中这样使用handler：



```java
public class CustomerActivity {
    // 省略部分代码
void createHandler() {
    new Handler() {
        @Override public void handleMessage(Message message) {
            super.handleMessage(message);
        }
    }.postDelayed(new Runnable() {
        @Override public void run() {
            while(true);
        }
    }, 1000);
}

      View mBtn = findViewById(R.id.h_button);
      mBtn.setOnClickListener(new View.OnClickListener() {
               @Override 
               public void onClick(View v) {
                          createHandler();
                         nextActivity();
                }
         })；
      // 省略部分代码
}
```

当Appl启动以后，framework会首先帮助我们完成UI线程的消息循环，也就是在UI线程中，Loop、MessageQueue、Message等等这些实例已经由framework帮我们实现了。所有的App主要事件，比如Activity的生命周期方法、Button的点击事件都包含在这个Message里面，这些Message都会加入到MessageQueue中去，所以，UI线程的消息循环贯穿于整个Application生命周期。因此当你在UI线程中生成Handler的实例，就会持有Loop以及MessageQueue的引用。

##### 2、解决方案：

> 将自定义的Handler和Runnable类生命成静态内部类，来解除和Activity的应用。

#### (五)、Sensor Manager导致的内存泄露:

##### 1、问题



```java
//省略部分代码
void registerListener() {
       SensorManager sensorManager = (SensorManager) getSystemService(SENSOR_SERVICE);
       Sensor sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ALL);
       sensorManager.registerListener(this, sensor, SensorManager.SENSOR_DELAY_FASTEST);
}

View smButton = findViewById(R.id.sm_button);
smButton.setOnClickListener(new View.OnClickListener() {
    @Override public void onClick(View v) {
        registerListener();
        nextActivity();
    }
});
//省略部分代码
```

通过Context调用的getSystemService获取系统服务，这些服务运行在他们自己的进程执行一系列后台工作或者提供和硬件交互的家口，如果Context对象需要在一个Service内部事件发生时随时受到通知，则需要把自己作为一个监听器注册进去，这样服务就会持有一个Activity。如果开发者忘记了在Activity被销毁钱注销这个监听器，就会导致内存泄露。

##### 2、解决方案

> 在onDestroy方法里面注销监听器。

#### (六)、检测内存泄露的工具:

[LeakCanary](https://link.jianshu.com?t=https://github.com/square/leakcanary)
 推荐使用LeakCanary来对内存泄露进行检测。LeakCanary是非常好用的第三方库，用来检测内存泄露，感兴趣的朋友可以去查阅LeakCannary的使用方法，使用它来检测App中内存泄露。

感谢:
 [https://juejin.im/post/5865bfa1128fe10057e57c63](https://link.jianshu.com?t=https://juejin.im/post/5865bfa1128fe10057e57c63)
 [http://blog.csdn.net/mr_liabill/article/details/49872527](https://link.jianshu.com?t=http://blog.csdn.net/mr_liabill/article/details/49872527)
 [http://www.cnblogs.com/xgjblog/p/5462417.html](https://link.jianshu.com?t=http://www.cnblogs.com/xgjblog/p/5462417.html)
 [http://blog.csdn.net/feiduclear_up/article/details/47356289](https://link.jianshu.com?t=http://blog.csdn.net/feiduclear_up/article/details/47356289)
 [http://www.cnblogs.com/android100/p/Android-Context.html](https://link.jianshu.com?t=http://www.cnblogs.com/android100/p/Android-Context.html)
 [https://developer.android.com/reference/android/app/Activity.html](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/Activity.html)
 [http://www.jianshu.com/p/7e78c535f1f5](https://www.jianshu.com/p/7e78c535f1f5)
 [http://www.jianshu.com/p/f24707874b04](https://www.jianshu.com/p/f24707874b04)
 [http://www.jianshu.com/p/94e0f9ab3f1d](https://www.jianshu.com/p/94e0f9ab3f1d)
 [http://www.jianshu.com/p/994cd73bde53](https://www.jianshu.com/p/994cd73bde53)
 [http://blog.csdn.net/lmj623565791/article/details/40481055/](https://link.jianshu.com?t=http://blog.csdn.net/lmj623565791/article/details/40481055/)
 [http://blog.csdn.net/qinjuning/article/details/7310620](https://link.jianshu.com?t=http://blog.csdn.net/qinjuning/article/details/7310620)
 [http://blog.csdn.net/guolin_blog/article/details/47028975](https://link.jianshu.com?t=http://blog.csdn.net/guolin_blog/article/details/47028975)



作者：隔壁老李头
链接：https://www.jianshu.com/p/e6ce2d03f8f9
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
