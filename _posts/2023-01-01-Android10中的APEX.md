---
layout: post
title:  "Android 10中的APEX"
date:   2024-01-01 19:38:38
catalog:  true
tags:
    - android组件源码阅读 

---

> 简介：

# Android 10中的APEX

Android碎片化的问题除了好多厂商加了更符合国人土豪味的特性之外，其实还有一个更基础性的问题就是升级太慢。为啥子？

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGY3dWE2M3hNaWI4ckdYNlpoV3BpYUlyME1HOWgxWEt4Wmh0Y2RXYUlQR3Voa3o0R0tpYWlhNUk0cXppY2tKMXNaaWNTZGZoamd0T3RNSDY0b2cvNjQw?x-oss-process=image/format,png) 

谷歌是AOSP代码的亲爹。但是有个问题，它老人家把代码搞出来后呢，还不能给厂商使用。因为谷老大没有硬件—给其他玩家用的硬件。

在这个生态链里，谷老大之后是华为，高通，MTK这样的芯片厂商。这些芯片厂商有芯片（SOC啊，全套都有，就好像拎包入住的精装修的房子——为毛码农可以拿房产做比喻，潘石屹不能学Python？），再加上自己偏底层，偏硬件的代码，配合谷老大的上层，组合一下卖给下游厂商。

接下来是华米OV魅，他们肯定是基于芯片厂商的代码来再做自己的工作。

虽然谷歌用了很多办法来提高新系统的适配速度（有技术上的，有厂商合作方面的），但这个长长的链条摆在这，新系统普及率比ios差得不是一星半点。痛定思痛，谷歌终于发现瓶颈在哪了，其实还是技术上的：

刚开始以为是驱动更新不及时，所以谷歌搞了Treble。从Android O开始。通过Treble，OS升级中和硬件相关的地方就尽可能摘出来，老的驱动也能跑在新的OS上。不知道用Treble后，OS升级率提高到多少，但显然还没让人满意。

Android Q，谷老大搞了一个mainline计划，这个计划中文名可以称作霸王硬上弓——霸王是谷歌，硬上弓也指这活很需要技术含量，不是谁都能搞。支撑mainline计划的核心就是本篇要讲的APEX。这么霸王的名字居然是Android Pony EXpress的简称（Android小马快线？）。

APEX和APK类似，它把Framework层中那些关键的东西搞成一个个的模块，然后可以单独升级这些模块。这些模块和就和一个一个的APK类似，就是一个压缩包，后缀名叫.apex。来看官方文档中对.apex文件格式的描述：

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGY3dWE2M3hNaWI4ckdYNlpoV3BpYUlyMEtERG9kbnBBRW1xeHFTWkhHQTFxSHFuRUJQY0VwaWNER2RleEJHTlVnUFZxQngxaWFxbXJTaHJ3LzY0MA?x-oss-process=image/format,png) 

apex和apk类似，实际上也是一个压缩文件。只不过apk和apex的目标不一样。请大家谨记

apk是应用程序的载体，对应用开发者而言，可以apk方式对应用功能进行升级。

apex是系统功能的载体，对系统开发者（目前看主要是谷歌）而言，可以apex方式对系统功能进行升级。这些apex包将来就发布在谷歌的playstore上供我们下载。简直不能太棒！

apex相当于对系统功能进行了更细粒度的划分，可以独立升级这些功能。而从另外某个意义上来说，这种做法也更加限制了设备厂商的魔改行为——这就是谷歌mainline计划的目的。

现在，我们可以把apex看成是一个一个的系统升级包，接下来我们看看系统中有哪些模块被划归到apex里了。

了解一下，有哪些apex包？

我们以模拟器+x86_64不带谷歌服务的镜像为例，启动模拟器看一下。

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGVmUzlTOHE2UVV2UG5ad0U0S2t6dDNvSG5DNnB3WFppYzJHM21JUG16T0NmVnBveHlZc2xPb3NXRGljcTJZd1NZd0YzdDVvSEVGYUt4QS82NDA?x-oss-process=image/format,png) 

/system目录下多了一个apex目录。这里存的就是可以通过apex方式升级的系统功能。现在先不谈怎么个升级法。先看看哪些功能被封装到不同的apex里了。我们看几个主要的。

com.android.runtime.debug

其核心是com.android.runtime。什么是runtime呢？就是ART虚拟机相关的东西。下面是这个apex包的成员：

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGVmUzlTOHE2UVV2UG5ad0U0S2t6dDNDN1pEZnZYMEdEeE83UHM1ZVN2MVRJWnJ3OWZYSHpTM004ZklWdjZIMklLQWxwQXRrVmFFTXcvNjQw?x-oss-process=image/format,png) 

图中，每一个apex包（或者是目录）都包含：

apex_manifest.json：这个是apex包的清单文件，类似apk中的AndroidManifest.xml。apex_manifest.json功能比AndroidManifest.xml要复杂，里边还包含该apex安装前和安装后要执行的命令。

apex_pubkey：apex包的签名信息。和apk一样，我们对apex包也需要校验

剩下的目录就是此apex包的实际内容。显然，什么so，可执行程序，系统级jar包都可以包含在apex包中

那么，com.android.runtime apex包都有哪些内容呢？

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGVmUzlTOHE2UVV2UG5ad0U0S2t6dDNmbERjUFhjUVcxOWc5NUd2dDB2bnZITXlRQlhIak5xdEtpYzF1VTZ3ZjF6M2ljaWI2OGtkdjRWaEEvNjQw?x-oss-process=image/format,png) 

以上是runtime apex的主要内容。可看出，它包含的大部分是ART虚拟机相关的so和可执行程序。另外，bionic C库的libc.so，libdl.so，libm.so也在其中。

另外，位于/system/lib下的libc.so等也变成了链接，指向位于apex包中的这些so。来看图：

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGVmUzlTOHE2UVV2UG5ad0U0S2t6dDNoSHoyS1ptM2tnc2FuVTdQRWtlSVphcm5UZ3owTHNnZlhiSUhteVF1U2JXWkhxMlUxQ2RqSFEvNjQw?x-oss-process=image/format,png) 

注意上图中的/apex/com.android.runtime目录，这个和apex包的处理逻辑有关，我们稍后介绍。

com.android.media

com.android.media是多媒体相关的apex包，直接看它的内容：

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGVmUzlTOHE2UVV2UG5ad0U0S2t6dDM4MFl6dFNjM3N0em1pY2paSHRIN2ZCZTIzQm1aYjZaNmRsZXdpY2RBaFk5cDJTSkNWVGhMVTBxUS82NDA?x-oss-process=image/format,png) 

media apex包还包含了libbinder.so，libc++.so等非常关键的so。但要指出的是，/system/lib下的libc++.so却不是链接到media下的libc++.so。也就是说，系统里存在多份libc++.so。一个是/system/lib下的libc++.so，另外一个是media apex下的libc++.so（实际上，另外一个apex包，com.android.media.swcodec下还有一个libc++.so）。

除了libc++.so外，其他好几个so也存在同样的情况。对这件事，我第一感觉是有点奇怪。为了加载到正确的so，可能需要链接器linker程序做对应的修改。此问题我没有继续查下去了，欢迎有知道的童鞋指教。

了解一下，apex的安装

apex的安装（包括更新，回退）是一个比较复杂的过程。

为此，谷歌不惜对init做了重大改动。这里，我不打算对init做更详细的分析，先浮光掠影带大家看一下。有需要的话可自己看看。

总体来说，init大体上还是我八年前在《深入理解Android 卷1》里分析的那个init的样子。比如都是解析init.rc文件，然后执行对应的动作。但我感觉现在的init对初学者非常的不友好。因为代码逻辑比较复杂，而且用上了C++（应该在Android 10之前就用上C++了）。

整体来说，10.0中的init将分为三个阶段执行。这三个阶段挺有意思，都是执行init，但传的参数不一样。

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGYyaWMwZVR2STAxRjg3dmpHUWNpY3VLYWM1Q003Ykx3MzJDM1Z1UjJQVm12UmVXUnB4SXdtMlZJQ2NadWljREo0cmY3S1lxZlQ1TEJFMWcvNjQw?x-oss-process=image/format,png) 

上图是init的main函数，稍微介绍下：

首先是kernel启动后执行不带参数的init进程，这就进到上图中的①，FirstStageMain。FirstStageMain里边将做android verified boot，也就是把上篇文章（了解一下，Android 10中镜像文件的制作）里的分区挂载上来。第一阶段的代码中有大量和分区，avb有关的内容。是一个比较好的了解相关知识的地方

FirstStageMain最后会

execv("/system/bin/init","selinux_setup")再次执行init（参数为"selinux_setup"）。这将进入图中的②。SetupSelinux阶段。这个阶段也就是selinux相关处理。selinux是一个完备的知识体系

（https://blog.csdn.net/Innost/article/details/19299937）。这几篇文章秉承我们深入理解抄底式的学习风格，当年我拿它在SONY移动做内部培训用。唯一的小遗憾是没太讲selinux的programming。因为我当初理解它selinux更多是配置的事情。现在看也不完全对。

SetupSelinux最后

execv("/system/bin/init","second_stage")，此后将转入图中的③。SecondStageMain就是我们早期init功能的处理之处。这块功能基本上用C++语言全部重写了（对比我在深入理解Android 卷1里2.3版本的init）。我个人感觉是init不美了。以前基于C结构体的OOP我觉得看着挺舒服....

apex作为一个系统功能安装包，肯定有一个对应的服务进行管理。在Android 10中，这个服务就是apexd。代码位于/system/apex下。init通过apexd.rc启动apex相关服务。看一下这个apexd.rc文件

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGYyaWMwZVR2STAxRjg3dmpHUWNpY3VLYXM4b01ycVZXb0tLNTlpYVdpYlZLUmtBUEJ1NThoRGNLRWtXOVcxMnlXRk5DelpOVUxtaWFqVU9Xdy82NDA?x-oss-process=image/format,png) 

apexd.rc有两个服务，一个是apexd-boostrap，一个是apexd。其实对应执行的程序都是/system/bin/apexd，只不过bootstrap带了一个参数“--bootstrap”。上面两个服务中，先启动apexd-bootstrap，然后再启动apexd。下面是apexd的入口代码：

 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy9LeUJpYTNZZ3N5aGYyaWMwZVR2STAxRjg3dmpHUWNpY3VLYVZhSlhQaHNBUVR6c29LWUx2Y1hxMmlhZjlrbkE5SXJIeDJIa0VsZjdEWWZDZWZBaWJRV3BFYmZ3LzY0MA?x-oss-process=image/format,png) 

apexd-bootstrap是主要处理apex包的地方。包括校验签名，apex版本比较等。然后，它会将apex包(例如/system/apex/com.android.runtime.debug等）挂载到/apex目录下。使用的方法是mount的MS_BIND标志。

到此，我们对apex的了解就告一段落。对一般开发者而言，只要知道apex包是被谷老大用来更新系统级功能的就行。另外，代码中有一个GSI（Generic System Image），这个GSI镜像可能包含的就是这些谷歌希望由自己来控制更新的apex包。通过这种方式，厂商能改的地方将大大受限，而能访问谷歌playstore的人们也可以第一时间享受谷歌的系统更新（但是谷歌经常也挖坑，搞出好多系统bug）