---
layout: post
title:  "scanPackageDirtyLi函数名的含义"
date:   2016-07-30 20:30:00
catalog:  true
tags:
    - android组件介绍
   

---

> 基于Android 6.0源码剖析，本文涉及的相关源码：

在PMS的源码中除了LI结尾的方法，还有其他例如**LIF、LPw、LPr**结尾的函数，下面列举了一部分。

| 后缀 |                            方法名                            |
| :--: | :----------------------------------------------------------: |
|  LI  | collectCertificatesLI() installPackageLI() scanPackageLI() scanDirLI() |



从上面的注释中可以了解到PMS中有两个锁，**mPackages**和**mInstallLock**，这两个锁在PMS中非常重要。而LI、LIF、LPw、LPr这些方法，L指的是Lock，I和P就指的是mInstallLock和mPackages。最后面w表示writing，r表示reading，F表示Freeze。

在上面注释中也可以看到mInstallLock是单线程的，操作慢，所以在已经持有`mPackages`同步锁的时候，千万不要再请求`mInstallLock`同步锁。

在上面注释中也可以看到mInstallLock是单线程的，操作慢，所以在已经持有`mPackages`同步锁的时候，千万不要再请求`mInstallLock`同步锁。

**错误方式**

```
synchronized (mPackages) {
synchronized (mInstaller) {
    // 这种情况是绝对不允许的。因为Install的处理时间会很长，导致对mPackages锁住的时间加长，会使得其他对mPackages操作的请求处于长时间等待。
}
}
```

**正确方式**

```
synchronized (mInstaller) {
synchronized (mPackages) {
    // 这种情况是允许的。因为mPackages处理完之后，其他对mPackages操作的请求可以对mPackages处理，不需要等待太久。
    // 由于处理Install的时间本身很长，synchronized (mPackages)又较快，所以不会对原本长时间持有mInstaller锁的情况有大的影响。
}
}
```

许多内部方法依赖于调用者来持有适当的锁，在调用函数时，要遵循以下规则

|    方法名    |                           使用方式                           |
| :----------: | :----------------------------------------------------------: |
| **fooLI()**  |          调用fooLI()，必须先持有`mInstallLock`锁。           |
| **fooLIF()** | 调用fooLIF()，必须先持有`mInstallLock`锁，并且正在被修改的包（package）必须被冻结（be frozen）。 |
| **fooLPr()** | 调用fooLPr()，必须先持有`mPackages`锁，并且只用于**读**操作。 |
| **fooLPw()** | 调用fooLPw()，必须先持有`mPackages`锁，并且只用于**写**操作。 |

例如要调用`installPackageTracedLI`，必须先持有`mInstallLock`锁，因此，调用为：

```
synchronized (mInstallLock) {
         installPackageTracedLI(args, res);
}
```

再例如`verifyPackageUpdateLPr`

```
// writer
synchronized (mPackages) {
     ......
		if (!verifyPackageUpdateLPr(origPackage, pkg)) {
         // New package is not compatible with original.
         origPackage = null;
         continue;
    }
    ......
}
```

最后还有一个注解**`@GuardedBy`**用于标记哪些变量要用同步锁保护起来。

```
@GuardedBy("mPackages")
private boolean mDexOptDialogShown;

// Used for privilege escalation. MUST NOT BE CALLED WITH mPackages
// LOCK HELD.  Can be called with mInstallLock held.
//用于权限提升。不能在保持mPackages锁的情况下调用。可以在保持mInstallLock的情况下调用。
@GuardedBy("mInstallLock")
final Installer mInstaller;

@GuardedBy("mPackages")
final ArrayMap<String, PackageParser.Package> mPackages =
        new ArrayMap<String, PackageParser.Package>();
```

这个注解表示在使用变量的地方，必须获取对应的锁，以防止并发错误

```
synchronized (mPackages) {
    dexOptDialogShown = mDexOptDialogShown;
}
```

嗯，又学习到了一个知识点。

Q android framework的代码中，有很多函数的名字都有Locked后缀，这Locked是什么意思?

A 不是线程安全，恰恰说明是非线程安全。说明这些函数需要在加锁的代码中调用。你找下调用位置就明白了。