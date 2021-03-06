---
layout:     post
title:      android插件的原理分析和简单实现
subtitle:   
date:       2019-04-21
author:     Jambooid
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - android插件技术
---

#### 背景

**插件化是安卓高级开发很难避开的一个模块，但是插件化的运用却通常只在一线的互联网公司中，导致很多小公司的开发没有什么机会去接触到它，很多小公司可能会用热修复，却不一定会去用插件化。**

插件化开发是将整个app拆分成多个模块，这些模块包括一个宿主和多个插件，每个模块都是一个apk，最终打包的时候宿主apk和插件apk分开打包。用户在使用宿主apk的时候可以动态的去下载插件apk，然后使用该功能而不是去更新整个app后才能使用新功能。

今天我就带大家实现以下最简单的插件化，首先是原理介绍：

#### 原理介绍

在实现插件化的过程中我们需要熟悉一些知识点，比如类加载机制，反射机制，apk的包结构，dx工具等。

为什么要学类加载机制呢，因为我们的apk是个插件，所以我们需要用我们的类加载器去加载插件中类，然后使用该类；

为什么要用到反射呢，我们把插件apk下载到本地后，肯定不希望每次使用它都得去从插件中获取，我们需要把它合并到宿主的app当中，然后就想使用正常的App一样去使用该功能，所以需要使用到反射去合并它们，当然反射的运用远远不止如此；

为什么要熟悉Apk的包结构和dx工具，很简单，只有熟悉Apk包结构才能了解dex文件；然后使用dx文件去生成dex文件。

##### 类加载机制

首先了解下一个类的生命周期，先来看下面这张图：

![img](https://s2.loli.net/2021/12/09/Jt65K3YNkjl7sfM.png)

image.png

类的生命周期可以分为加载，连接（验证，准备，解析），初始化，使用，卸载。这里我们主要讲一下加载过程。

在类的加载阶段，主要完成以下三个过程：

1. 通过一个类的全限定名来获取此类的二进制流。
2. 将这个字节流所代表的静态存储结构转换为方法去的运行时数据结构。
3. 在内存中生成一个代码这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

从哪里将一个类加载成二进制流的方式有：

- 从zip包中读取，这就发展成了我们常见的JAR、AAR依赖。
- 运行时动态生成，这是我们常见的动态代理技术，在java.reflect.Proxy中就是用ProxyGenerateProxyClass来为特定接口生成代理类的二进制流。

说到加载，不得不提到类加载器，下面就讲述以下**Java类加载器和Android类加载器**

###### Java类加载器

类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远远不限于类的加载阶段。对于任意一个类，都需要由它的类加载器和这个类本身一同确定其在就Java虚拟机中的唯一性，也就是说，即使两个类来源于同一个Class文件，只要加载它们的类加载器不同，那这两个类就必定不相等。这里的“相等”包括了代表类的Class对象的`equals()、isAssignableFrom()、isInstance()`等方法的返回结果，也包括了使用instanceof关键字对对象所属关系的判定结果。

**站在Java虚拟机的角度来讲，只存在两种不同的类加载器**：

- 启动类加载器：它使用C++实现（这里仅限于Hotspot，也就是JDK1.5之后默认的虚拟机，有很多其他的虚拟机是用Java语言实现的），是虚拟机自身的一部分。
- 所有其他的类加载器：这些类加载器都由Java语言实现，独立于虚拟机之外，并且全部继承自抽象类java.lang.ClassLoader，这些类加载器需要由启动类加载器加载到内存中之后才能去加载其他的类。

**站在Java开发人员的角度来看，类加载器可以大致划分为以下三类**：

- 启动类加载器：Bootstrap ClassLoader，跟上面相同。它负责加载存放在JDK\jre\lib(JDK代表JDK的安装目录，下同)下，或被-Xbootclasspath参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar，所有的java.*开头的类均被Bootstrap ClassLoader加载）。启动类加载器是无法被Java程序直接引用的。
- 扩展类加载器：Extension ClassLoader，该加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载JDK\jre\lib\ext目录中，或者由java.ext.dirs系统变量指定的路径中的所有类库（如javax.*开头的类），开发者可以直接使用扩展类加载器。
- 应用程序类加载器：Application ClassLoader，该类加载器由sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

**双亲委托机制**

双亲委托模型的工作流程非常的简单，如下所示：

![img](https://s2.loli.net/2021/12/09/rl8uaHGSodKOjx2.png)

image.png

如果一个类加载器收到了加载类的请求，它不会自己立即去加载类，它会先去请求父类加载器，每个层次的类加载器都是如此。层层传递，直到传递到最高层的类加载器，只有当父类加载器反馈自己无法加载这个类，才会有当前子类加载器去加载该类。



```java
public abstract class ClassLoader {

    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            //首先，检查该类是否已经被加载
            Class c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    //先调用父类加载器去加载
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    //如果父类加载器没有加载到该类，则自己去执行加载
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                }
            }
            return c;
    }
}

复制代码
```

这是为了保证Android中系统的最基础的行为，保证一些类比如Object只能是最顶层的类加载的。

###### Android类加载器

可以看到下面一共有8个ClassLoader相关类，其中有一些和Java中的ClassLoader相关类十分类似，下面简单对它们进行介绍：

![img](https://s2.loli.net/2021/12/09/wczvL3HdEXPiV1q.png)

image.png

- `ClassLoader`是一个抽象类，其中定义了ClassLoader的主要功能。
- `BootClassLoader`是ClassLoader的内部类，用于**预加载preload()**常用类，加载一些系统Framework层级需要的类，我们的Android应用里也需要用到一些系统的类等，**是Android平台上所有ClassLoader的最终parent**,这个内部类是包内可见,所以我们没法使用。
- SecureClassLoader类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。
  - `URLClassLoader`类和JDK8中的URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URl路径从jar文件和文件夹中加载类和资源。**在Android中基本无法使用**
- BaseDexClassLoader继承自ClassLoader，是抽象类ClassLoader的具体实现类，PathClassLoader和DexClassLoader都继承它。
  - `PathClassLoader`加载系统类和应用程序的类，如果是加载非系统应用程序类，则会加载data/app/目录下的dex文件以及包含dex的apk文件或jar文件
  - `DexClassLoader` 可以加载自定义的dex文件以及包含dex的apk文件或jar文件，也支持从SD卡进行加载
  - `InMemoryDexClassLoader`是Android8.0新增的类加载器，继承自BaseDexClassLoader，用于加载内存中的dex文件。

**这里需要注意一下`PathClassLoader`和`DexClassLoader`，在8.0之前，它们二者的唯一区别是第二个参数optimizedDiredtory，这个参数的意思是odex(优化的dex)存放的路径。在8.0及之后，二者就完全一样了，这里说的是Art运行时**。

**问题：PathClasLoader和BootClassLoder 分别加载什么类？**

PathClasLoader加载应用的类，BootClassLoder加载SDK的类（Framework），比如Activity的类加载器是BootClassLoader，而MainActivity，AppcompatActivity类的加载器是PathClassLoader。

##### 反射

反射就是在运行时采用知道操作的类是什么，并且可以在运行时获取类的完整构造，并调用对应的方法和属性，下面先来看看反射中常用的API。

![img](https://s2.loli.net/2021/12/09/hIBsFMcjDV1ERx3.png)

image.png

###### 反射为什么耗性能？

1. Method#invoke 方法会对参数做封装和解封操作
2. 需要检查方法可见性
3. 需要校验参数
4. 反射方法难以内联
5. JIT 无法优化
6. 调用的过程中会产生大量的临时对象
7. 影响反射效率的还有方法表查找，毕竟调用方法前先要确保找到找个方法。不同与普通方法的调用仅仅需要多态选择，反射调用的方法需要遍历方法表或父类方法表找到。

详细内容可以参考[大家都说 Java 反射效率低，你知道原因在哪里么](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.cn%2Fpost%2F6844903965725818887%23comment)，还有R大的[文章](https://links.jianshu.com/go?to=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Frednaxelafx.iteye.com%2Fblog%2F548536)。

##### Apk的包结构



```css
- META-INF
- res
   - anim
   - color
   - drawable
   - drawable-hdpi
   - drawable-land
   - drawable-land-hdpi
   - drawable-mdpi
   - drawable-port
   - drawable-port-hdpi
   - layout
   - layout-land
   - layout-port
   - xml
- AndroidManifest.xml
- classes.dex
- resources.arsc
复制代码
```

dex是Android系统的可执行文件，包含应用程序的全部操作指令以及运行时数据。由于dalvik是一种针对嵌入式设备而特殊设计的java虚拟机，所以dex文件与标准的class文件在结构设计上有着本质的区别。

##### dx工具的使用

当java程序编译成class后，还需要使用dx工具将所有的class文件整合到一个dex文件，目的是其中各个类能够共享数据，在一定程度上降低了冗余，同时也是文件结构更加经凑，实验表明，dex文件是传统jar文件大小的50%左右。

#### 最简单的插件化

先说该项目的大致思路，就是将一个Test.java(Test.class)生成一个dex文件，然后放到我们的虚拟机sd卡下面，然后通过类加载器去加载该Dex中的Test.java，最后调用该类的方法。这是一个**最简单的插件化思路**，没有动态下载文件，而且还是dex文件也不是zip文件，apk文件，文件存放在本地。

新建一个项目PluginGoDemo，然后在该项目下再新建一个plugin(module)，我们在plugin项目中创建我们的Test.java文件，目录结构如下



![img](https://s2.loli.net/2021/12/09/cMBPdnT6vUtwRul.png)

image.png

我们的Test.java代码如下：



```csharp
public class Test {

    public static void print(){
        System.out.println("============我是Jackie");
    }
}
复制代码
```

首先将plugin这个module的Test.class打包成dex文件，需要配置dx环境变量，在配置文件下配置



```bash
export PATH="${PATH}:/Users/jackie/Library/Android/sdk/build-tools/30.0.2"
```

然后输入打包命令



```kotlin
~/Desktop/WorkPlace/AndroidWorkPlace/PluginGoDemo/plugin/build/intermediates/javac/debug/classes/com/jackie/plugin » dx --dex --output=output.dex Test.class  

PARSE ERROR:
class name (com/jackie/plugin/Test) does not match path (Test.class)
...while parsing Test.class
1 error; aborting
```

出现这个问题的原因是路径是Test.class与Test.java里面的packagename不匹配，只需要把com包整个复制出来就ok了。把com包拷贝到任何目录下继续执行该命令，比如拷贝到`Desktop/Plugin`目录下：



```ruby
~ » cd Desktop/Plugin                                                                                       ~/Desktop/Plugin » dx --dex --output=output.dex com/jackie/plugin/Test.class
```

打包出我们想要的output.dex，然后将它push到Android虚拟机的SD卡下面



```ruby
~/Desktop/Plugin » adb push /Users/jackie/Desktop/Plugin/output.dex /sdcard/                                   
/Users/jackie/Desktop/Plugin/output.dex: 1 file pushed, 0 skipped. 3.1 MB/s (724 bytes in 0.000s)
```

##### 代码调用dex文件中的类



```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                //在android28（9.0）虚拟机上测试成功
                PathClassLoader pathClassLoader = new PathClassLoader("/sdcard/output.dex",null);
                try {
                    Class<?> clazz = pathClassLoader.loadClass("com.jackie.plugin.Test");
                    Method method = clazz.getMethod("print");
                    method.invoke(null);

                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

最后添加上权限

```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
复制代码
```

**注意，代码中并没有动态申请权限，需要我们在系统设置中手动开启sd卡的访问权限**。

最后打印出我们想要的结果

```dart
2020-11-25 17:10:27.290 4823-4823/com.jackie.plugingodemo E/linker: library "/storage/emulated/0/oat/x86/output.odex" ("/storage/emulated/0/oat/x86/output.odex") needed or dlopened by "/system/lib/libart.so" is not accessible for the namespace: [name="(default)", ld_library_paths="", default_library_paths="/system/lib", permitted_paths="/system/lib/drm:/system/lib/extractors:/system/lib/hw:/system/product/lib:/system/framework:/system/app:/system/priv-app:/vendor/framework:/vendor/app:/vendor/priv-app:/odm/framework:/odm/app:/odm/priv-app:/oem/app:/system/product/framework:/system/product/app:/system/product/priv-app:/data:/mnt/expand"]
2020-11-25 17:10:27.290 4823-4823/com.jackie.plugingodemo I/ie.plugingodem: The ClassLoaderContext is a special shared library.
2020-11-25 17:10:27.292 4823-4823/com.jackie.plugingodemo I/System.out: ============我是Jackie
```

##### 合并Dex文件

合并dex，也就是将我们dex和apk的class.dex合并，都放到宿主dexElements数组中。App每次启动从该数组中加载。

```dart
public static void loadClass(Context context){

        //合并dexElements

        try{
            //获取BaseDexClassLoader中的pathList（DexPathList）
            Class<?> clazz = Class.forName("dalvik.system.BaseDexClassLoader");
            Field pathListField = clazz.getDeclaredField("pathList");
            pathListField.setAccessible(true);

            //获取DexPathList中的dexElements数组
            Class<?> dexPathListClass = Class.forName("dalvik.system.DexPathList");
            Field dexElementsField = dexPathListClass.getDeclaredField("dexElements");
            dexElementsField.setAccessible(true);

            //宿主的类加载器
            ClassLoader pathClassLoader = context.getClassLoader();
            //DexPathList类的对象
            Object hostPathList = pathListField.get(pathClassLoader);
            //宿主的dexElements
            Object[] hostDexElements = (Object[]) dexElementsField.get(hostPathList);

            String apkPath = "/sdcard/output.dex";
            //插件的类加载器
            ClassLoader dexClassLoader = new DexClassLoader(apkPath
                    ,context.getCacheDir().getAbsolutePath()
                    ,null
                    ,pathClassLoader);

            //DexPathList类的对象
            Object pluginPathList = pathListField.get(dexClassLoader);
            //宿主的dexElements
            Object[] pluginElements = (Object[]) dexElementsField.get(pluginPathList);

            //创建一个新数组
            Object[] newDexElements = (Object[]) Array.newInstance(hostDexElements.getClass().getComponentType()
                    ,hostDexElements.length + pluginElements.length);
            System.arraycopy(hostDexElements,0,newDexElements,0,hostDexElements.length);
            System.arraycopy(pluginElements,0,newDexElements,hostDexElements.length,pluginElements.length);

            //赋值
            dexElementsField.set(hostPathList,newDexElements);

        }catch (Exception e){
            e.printStackTrace();
        }

    }

//调用测试
        loadClass(this);
        try {
            Class<?> clazz = Class.forName("com.jackie.plugin.Test");
            Method method = clazz.getMethod("print");
            method.invoke(null);
        }catch (Exception e){
            e.printStackTrace();
        }

复制代码
```

到此，最简单的插件化就完成了，喜欢的点个赞和关注吧~



作者：凤邪摩羯
链接：https://www.jianshu.com/p/586f3925eb82
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
