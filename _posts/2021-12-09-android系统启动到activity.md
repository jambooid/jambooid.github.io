---
layout: post
title:  "android系统启动到activity"
date:   2016-07-30 20:30:00
catalog:  true
tags:
    - android组件源码阅读
       

---

> 基于Android 6.0源码剖析，本文涉及的相关源码：

# 系统启动

从系统启动开始复习。我想起我当时的毕业设计，是第一次真正意义上的接触Linux系统。现在要把这些知识，从脑海中回忆整理出来。

Android系统的内核实际上是一个Linux内核，系统的一切启动，都是从Linux内核开始启动。

复习一下，Linux系统启动流程：
 Linux启动流程大致分为2个步骤：
 引导和启动。

##### 引导：

分为BIOS上电子检(POST)和引导装载程序（GRUB）

##### 启动

内核引导，内核init初始化，启动 systemd，其是所有进程之父。

## Android系统启动

这里Android启动流程系统稍微有点不一样。
 这里借别人的一幅图：



![img](https://s2.loli.net/2021/12/09/gPxfXAOGwkqvsrS.png)

启动流程.png

到了BootLoader内核引导程序的装载之后，会启动Linux内核。此时Linux内核就去启动第一个进程，init进程。

启动进程，那么一定会启动到init.cpp中的main函数。

文件：/[system](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2F)/[core](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2Fcore%2F)/[init](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2Fcore%2Finit%2F)/[init.cpp](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2Fcore%2Finit%2Finit.cpp)



```php
Parser& parser = Parser::GetInstance();
parser.AddSectionParser("service",std::make_unique<ServiceParser>());
parser.AddSectionParser("on", std::make_unique<ActionParser>());
parser.AddSectionParser("import", std::make_unique<ImportParser>());
parser.ParseConfig("/init.rc");
```

此时会通过Parser方法去解析当前目录下的init.rc文件中的语法。
 文件开头会配置下面这个文件：

```kotlin
import /init.${ro.zygote}.rc
```

```kotlin
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
```

此时rc的脚本，意思是service是启动一个服务名字是zygote，通过/system/bin/app_process。 此时为zygote创建socket，访问权限600，onrestrat是指当这些服务需要重新启动时，执行重新启动。更多相关可以去看看：/[system](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2F)/[core](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2Fcore%2F)/[init](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2Fcore%2Finit%2F)/[readme.txt](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fsystem%2Fcore%2Finit%2Freadme.txt)

我们转到/[frameworks](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2F)/[base](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2F)/[cmds](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcmds%2F)/[app_process](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcmds%2Fapp_process%2F)/[app_main.cpp](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcmds%2Fapp_process%2Fapp_main.cpp)

找到下个核心，Zygote启动主函数。

### Zygote（孵化进程）启动

Zygote从名字上就知道，这个初始进程的作用像一个母鸡一样，孵化所有的App的进程。

```cpp
int main(int argc, char* const argv[])
{
    ...
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    ...
    // Parse runtime arguments.  Stop at first unrecognized option.
//解析命令
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
...
//设置启动模式
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

实际上这里主要分两步：
 第一步：解析刚才传进来的配置。
 第二步：根据配置来决定app_main启动模式

根据我们的命令我们可以知道系统启动调用的命令会使得zygote标识位为true。

此时进入到ZygoteInit里面初始化。在这之前，我们看看这个runtime对象在start方法做了什么事情。AppRuntime继承于AndroidRuntime

文件：/[frameworks](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2F)/[base](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2F)/[core](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2F)/[jni](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2F)/[AndroidRuntime.cpp](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2FAndroidRuntime.cpp)

#### AndroidRuntime::start

```cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");

    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }
//确认Android目录环境
    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);
//启动虚拟机
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

//注册jni方法
    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

//准备好环境之后，为传入的选项转化为java 层的String对象
    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

//根据传进来的class name,反射启动对应的类
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

这个方法实际上包含了极大量的工作。此时我们能看到的是Zygote从app_main的入口想要启动ZyogteInit的类。我们看到这个签名就知道这是Android系统启动以来第一个要加载的Java类。那么此时我们必须保证虚拟机启动完成。

所以AndroidRunTime要进行的步骤一共为三大步骤：

##### 1.初始化虚拟机

##### 2.注册jni方法

##### 3.反射启动ZygoteInit。

### 1.初始化虚拟机

```cpp
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
```

我们来研究看看这几个方法究竟做了啥。
 先看看jni_invocation的初始化方法

```cpp
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
```



```cpp
static const char* kLibrarySystemProperty = "persist.sys.dalvik.vm.lib.2";
static const char* kDebuggableSystemProperty = "ro.debuggable";
#endif
static const char* kLibraryFallback = "libart.so";

bool JniInvocation::Init(const char* library) {
#ifdef __ANDROID__
  char buffer[PROP_VALUE_MAX];
#else
  char* buffer = NULL;
#endif
  library = GetLibrary(library, buffer);
  // Load with RTLD_NODELETE in order to ensure that libart.so is not unmapped when it is closed.
  // This is due to the fact that it is possible that some threads might have yet to finish
  // exiting even after JNI_DeleteJavaVM returns, which can lead to segfaults if the library is
  // unloaded.
  const int kDlopenFlags = RTLD_NOW | RTLD_NODELETE;
  handle_ = dlopen(library, kDlopenFlags);
  if (handle_ == NULL) {
    if (strcmp(library, kLibraryFallback) == 0) {
      // Nothing else to try.
      ALOGE("Failed to dlopen %s: %s", library, dlerror());
      return false;
    }
    // Note that this is enough to get something like the zygote
    // running, we can't property_set here to fix this for the future
    // because we are root and not the system user. See
    // RuntimeInit.commonInit for where we fix up the property to
    // avoid future fallbacks. http://b/11463182
    ALOGW("Falling back from %s to %s after dlopen error: %s",
          library, kLibraryFallback, dlerror());
    library = kLibraryFallback;
    handle_ = dlopen(library, kDlopenFlags);
    if (handle_ == NULL) {
      ALOGE("Failed to dlopen %s: %s", library, dlerror());
      return false;
    }
  }
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetDefaultJavaVMInitArgs_),
                  "JNI_GetDefaultJavaVMInitArgs")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_CreateJavaVM_),
                  "JNI_CreateJavaVM")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetCreatedJavaVMs_),
                  "JNI_GetCreatedJavaVMs")) {
    return false;
  }
  return true;
}
```

核心方法实际上只有一处：

```dart
library = GetLibrary(library, buffer);
handle_ = dlopen(library, kDlopenFlags);
```

dlopen是指读取某个链接库。这个library是通过getlibrary获得的。这个方法就不贴出来了。这个方法解释就是通过读取kLibrarySystemProperty下的属性，来确定加载什么so库。而这个so库就是我们的虚拟机so库。

如果对那些做过编译Android机子的朋友就能明白，实际上这个属性在/art/Android.mk中设定的。里面设定的字符串一般是libart.so或者libdvm.so.这两个分别代表这是art虚拟机还是dvm虚拟机。

当然这是允许自己编写虚拟机，自己设置的。因此，在这个方法的最后会检测，这个so是否包含以下三个函数：

1. JNI_GetDefaultJavaVMInitArgs
2. JNI_CreateJavaVM
3. JNI_GetCreatedJavaVMs

只有包含这三个函数，Android初始化才会认可这个虚拟机做进一步的初始化工作。

此时我们已经初始化好了art虚拟机（此时是Android 7.0），我们要开始启动虚拟机了。

### 2.启动化虚拟机

```kotlin
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
```

在startVm中做了两个工作，第一个把对虚拟机设置的参数设置进去，第二点，调用虚拟机的JNI_CreateJavaVM方法。核心方法如下：

```kotlin
    /*
     * Initialize the VM.
     *
     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
     * If this call succeeds, the VM is ready, and we can start issuing
     * JNI calls.
     */
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }

    return 0;
```

此时如果创建成功，返回的int是大于等于0，小于0则是创建异常。并且把初始化好之后的数据类型赋值给JavaVm，JniEnv。

而下面这个函数onVmCreated(env);

```ruby
void AndroidRuntime::onVmCreated(JNIEnv* env)
{
    // If AndroidRuntime had anything to do here, we'd have done it in 'start'.
}
```

是不做任何处理的。这里等下说，还有什么妙用。

### 2.注册jni方法



```kotlin
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
```

我们看看这个注册是做什么的。

```php
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives"); 
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```

实际上这个核心方法是register_jni_procs。注册jni方法。

```cpp
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
            return -1;
        }
    }
    return 0;
}
```

RegJNIRec这个是一个结构体。mProc就是每个方法的方法指针：每一次都会调用注册这个数组中的方法。我们看看这个array的数组都有什么。

```cpp
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
  ...
    REG_JNI(register_com_android_internal_os_Zygote),
    REG_JNI(register_com_android_internal_util_VirtualRefBasePtr),
    REG_JNI(register_android_hardware_Camera),
    REG_JNI(register_android_hardware_camera2_CameraMetadata),   
    REG_JNI(register_android_hardware_SoundTrigger),
    REG_JNI(register_android_hardware_UsbDevice),
    REG_JNI(register_android_hardware_UsbDeviceConnection),
    REG_JNI(register_android_hardware_UsbRequest),
    ....
};
```

看到了把。这些方法无一不是我们熟悉的方法，什么Bitmap，Parcel，Camera等等api，这些需要调用native的类。

这就说回来了，onVmCreated这个留下的空函数做什么的。一般我们开发留下一个空函数一般不是兼容版本，就是交给用户做额外处理。实际我们开发jni的时候，我们可以通过这个方法，常做的两件事情，第一，创建一个JniEnv并为其绑定线程，第二，我们可以在这个时刻做好方法映射，让我们内部的方法，不需要遵守jni的命名规则（包+方法名），这样能够大大的扩展我们so的编写的可读性和复用。

### 3.反射启动ZygoteInit



```php
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
        }
    }
```

此时就走到反射调用ZygoteInit流程，此时传进来的slashClassName就是ZygoteInit的包名。

## ZygoteInit

此时进入到ZygoteInit的类中。我们看看main方法

文件：/[frameworks](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2F)/[base](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2F)/[core](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2F)/[java](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2F)/[com](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2F)/[android](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2F)/[internal](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2Finternal%2F)/[os](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2Finternal%2Fos%2F)/[ZygoteInit.java](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2Finternal%2Fos%2FZygoteInit.java)



```tsx
public static void main(String argv[]) {
        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        try {
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygoteInit");
            RuntimeInit.enableDdms();
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
//解析传进来的命令，主要是确定是否要启动SystemServer
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
//注册监听
            registerZygoteSocket(socketName);
//准备资源
            preload();
//装载好虚拟机之后，做一次gc
           gcAndFinalize();
          ...

            // Zygote process unmounts root storage spaces.
            Zygote.nativeUnmountStorageOnInit();

            ZygoteHooks.stopZygoteNoThreadCreation();
//启动SystemServer
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }
//启动循环
            Log.i(TAG, "Accepting command socket connections");
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```

这里做的事情主要是三步：
 第一步：解析命令是否要启动SystemServer
 第二步：registerZygoteSocket来打开socket监听命令，是否孵化新的进程
 第三步：启动SystemServer
 第三步：runSelectLoop 启动一个系统的looper循环

### 启动SystemServer

```java
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
...
        ZygoteConnection.Arguments parsedArgs = null;

 String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
//启动新的进程forkSystemServer
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

这个方法分两步：
 1.fork出SystemServer进程
 2.初始化SystemServer

##### 1.fork出SystemServer进程

这个步骤的核心方法是fork出SystemServer进程
 我们稍微往这个方法下面看看，究竟有什么问题？



```cpp
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        // Enable tracing as soon as we enter the system_server.
       if (pid == 0) {
            Trace.setTracingEnabled(true);
        }
        VM_HOOKS.postForkCommon();
}
```

核心方法是调用native方法进行fork。
 文件[com_android_internal_os_Zygote.cpp](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fcom_android_internal_os_Zygote.cpp)

```cpp
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL);
  if (pid > 0) {
      // The zygote process checks whether the child process has died or not.
      ALOGI("System server process %d has been created", pid);
      gSystemServerPid = pid;
      // There is a slight window that the system server process has crashed
      // but it went unnoticed because we haven't published its pid yet. So
      // we recheck here just to make sure that all is well.
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
      }
  }
  return pid;
}
```

ForkAndSpecializeCommon这个方法就是所有进程fork都会调用这个这个native函数。这个函数实际上是调用linux的fork函数来拷贝出新的进程。

##### 2.初始化SystemServer进程

```undefined
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }
```

孵化好新的进程。这里返回结果。加入结果pid是等于0.说明会走这个分支。我们这里只看一个孵化进程的情况。看看这个方法handleSystemServerProcess(parsedArgs);

```java
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {

        closeServerSocket();
...
            /*
             * Pass the remaining arguments to SystemServer.
             */
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }

        /* should never reach here */
    }
```

由于这个新的进程不需要接收消息，去孵化进程，所以第一件事情就是关闭socket的监听。第二件事就是通过RuntimeInit初始化SystemServer。

文件：[com](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2F)/[android](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2F)/[internal](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2Finternal%2F)/[os](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2Finternal%2Fos%2F)/[RuntimeInit.java](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F7.0.0_r1%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fcom%2Fandroid%2Finternal%2Fos%2FRuntimeInit.java)

```java
    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();

        commonInit();
        nativeZygoteInit();
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```

此时nativeZygoteInit会初始化我们ProcessState时候，同时初始化我们鼎鼎大名的Binder驱动。

我们看看核心方法applicationInit做了啥：

```java
   private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
...
        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```

就是这个方法把传进来的参数初始化，把之前“com.android.server.SystemServer”参数反射其main方法。



```cpp
   public static void main(String[] args) {
        new SystemServer().run();
    }
```

这里初始化“android_servers”的so库，里面包含了大量的Android的native需要加载的native服务，以及我们经常会用到各种服务（Display，powermanager，alarm，inputmanager,AMS,WMS）。创建了我们熟悉的SystemServerManger，为这个进程创建了Looper。

### runSelectLoop启动一个Zygote的循环。

```csharp
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
//sServerSocket中文件描述符的集合
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);
//等待事件的循环
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
//每一次执行完了动作之后，等到又新的进程进来等待唤醒
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
//执行的核心
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```

这是构建的一段死循环。可以说是系统主Loop，就像iOS，ActivityThread中的Looper，用来响应外界的事件。

这里面简单的逻辑如下，一旦有socket中监听到了数据过来，将会执行runOnce()方法，接着把对应的socket等待处理事件以及，对应的socket文件符移除掉。此时事件队列为0，这个循环再一次把Zygote监听添加回来。

#### runOnce

runOnce做的事情分为三部分：

- 1. 调用readArgumentList读取socket传过来的数据
- 1. 通过 forkAndSpecialize fork新进程
- 1. 处理fork出来的子进程

父进程关闭fd入口，同时设置好子进程的pid。而fork出来的子进程将会在下篇继续

这里按照惯例，我们画一幅时序图，总结

![img](https://s2.loli.net/2021/12/09/jLVY4A5ByD2Munw.png)

Android系统启动.png

### zygote 客户端

实际上一般的ZygoteSocket的客户端，一般为SystemServer中的ActivitymanagerService.

我们看看在Android 7.0中当不存在对应的应用进程时候，会调用startProcessLocked方法中Process的start方法。
 最终会调用

```java
 public static final String ZYGOTE_SOCKET = "zygote";


    private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }
```

这里的核心会调用一次ZygoteState的connect方法。

```java
        public static ZygoteState connect(String socketAddress) throws IOException {
            DataInputStream zygoteInputStream = null;
            BufferedWriter zygoteWriter = null;
            final LocalSocket zygoteSocket = new LocalSocket();

            try {
                zygoteSocket.connect(new LocalSocketAddress(socketAddress,
                        LocalSocketAddress.Namespace.RESERVED));

                zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());

                zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                        zygoteSocket.getOutputStream()), 256);
            } catch (IOException ex) {
                try {
                    zygoteSocket.close();
                } catch (IOException ignore) {
                }

                throw ex;
            }

            String abiListString = getAbiList(zygoteWriter, zygoteInputStream);
            Log.i("Zygote", "Process: zygote socket opened, supported ABIS: " + abiListString);

            return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                    Arrays.asList(abiListString.split(",")));
        }
```

此时会尝试的通过zygoteSocket也就是LocalSocket 去连接名为zygote的socket。也就是我们最开始初始化的在ZygoteInit中registerZygoteSocket的socket名字。

调用connect方法，唤醒Os.poll方法之后，再唤醒LocalServerSocket.accept方法，在循环的下一个，调用runOnce。

那么zygote又是怎么启动ActivityThread，这个应用第一个启动的类呢？

第一次看runOnce代码的老哥可能会被这一行蒙蔽了：

```java
ZygoteConnection newPeer = acceptCommandPeer(abiList);
```

实际上在ZygoteConnection中，这个abiList不起任何作用。真正起作用的是ZygoteConnection.runOnce中readArgumentList
 方法。 

实际上所有的字符串都是通过zygote的SocketReader读取出来，再赋值给上层。进行fork出新的进程。
 在ActivityManagerService的startProcessLocked

```csharp
if (entryPoint == null) entryPoint = "android.app.ActivityThread";

    Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
```

第一个参数就是ActivityThread，通过start方法，来打runOnce之后，进去handleChildProc，把ActivityThread的main反射出来，开始了Activity的初始化。而实际上Process.start方法就是一个socket往Zygote中写数据。

#### handleChildProc

因此，当fork之后，我们继续回到ZygoteInit的handleChildProc子进程处理。

```java
private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {
        closeSocket();
        ZygoteInit.closeServerSocket();

        if (descriptors != null) {
            try {
                Os.dup2(descriptors[0], STDIN_FILENO);
                Os.dup2(descriptors[1], STDOUT_FILENO);
                Os.dup2(descriptors[2], STDERR_FILENO);

                for (FileDescriptor fd: descriptors) {
                    IoUtils.closeQuietly(fd);
                }
                newStderr = System.err;
            } catch (ErrnoException ex) {
                Log.e(TAG, "Error reopening stdio", ex);
            }
        }

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        // End of the postFork event.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }
```

子进程将关闭socket，关闭socket的观测的文件描述符。这里就能完好的让进程的fdtable(文件描述符表)腾出更多的控件。接着走RuntimeInit.zygoteInit.接下来的逻辑就和SystemServer一样。同样是反射了main方法，nativeZygoteInit 同样会为新的App绑定继承新的Binder底层loop,commonInit 为App的进程初始化异常处理事件。我们可以来看看ActivityThread中的main方法。

```java
public static void main(String[] args) {
...

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

在这里面初始化了ActivityThread对象，并且传入attach方法。

#### ActivityThread的绑定

```java
private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }           
        } else {
...
        }   
    }
```

实际上这里做的事情核心有两个:

- 第一个把ApplictionThread绑定到AMS，让之后我们startActivity能够通过这个Binder对象找到对应的方法，从而正确的执行Activity中的正确的生命周期。同时为Binder添加gc的监听者。Binder的详细情况会在Binder解析中了解到
- 第二个就是为ViewRootImpl设置内存管理。这个类将会在后的view的绘制了解到。

至此，从Linux内核启动到应用的AcivityThread的大体流程就完成了。

上个图总结：

![img](https://s2.loli.net/2021/12/09/4Dcfe1C6gMojQlV.png)

zygote通信原理.png

作者：yjy239
链接：https://www.jianshu.com/p/e5231f99f2a1
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。