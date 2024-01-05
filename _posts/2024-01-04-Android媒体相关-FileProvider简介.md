---
layout: post
title:  "Android媒体相关-FileProvider简介"
catalog:  true
tags:
    - android组件源码阅读



---

# FileProvider简介

Android 7.0 在应用间共享文件
对于面向Android 7.0的应用，Android框架执行的StrictMode API政策禁止在您的应用外部公开file://URI，要在应用间共享文件，您应发送一项content://URI，并授予URI临时访问权限。进行此授权的最简单方式是使用FileProvider类。

## FileProvider简介

FileProvider是ContentProvider的一个特殊的子类，它让应用间共享文件变得更加容易，其通过创建一个Content URI来代替File URI。

一个Content URI 允许开发者可赋予一个临时的读或写权限。当创建一个包含Content URI的Intent的时候，为了能够让另一个应用也可以使用这个URI，你需要调用Intent.setFlags()来添加权限。只要接收Activity的栈是活跃的，则客户端应用就可以获取到这些权限。如果该Intent是用来启动一个Service，则只要该Service是正在运行的，则也可以获取到这些权限。

通过Content URI来提高文件访问的安全性，使得FileProvider成为Android安全基础设置的一个关键部分。

## FileProvider的使用

1. ### 定义一个FileProvider

```xml
<manifest>
    ...
    <application>
        ...
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.mydomain.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            ...
        </provider>
        ...
    </application>
</manifest>
```

2. ### 指定有效的文件

  需要在XML文件中指定其存储的路径。使用<paths>标签。name属性表示在URI中的描述(隐藏真实路径)，path属性表示文件实际存储的位置。

res/xml/file_paths.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <paths>
        <files-path name="my_files" path="text/"/>
        <cache-path name="my_cache" path="text/" />
        <external-files-path name="external-files-path" path="text/" />
        <external-path name="my_external_path" path="text/" />
    </paths>
</resources>
```

AndroidManifest.xml

```xml
    <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="com.ysx.fileproviderserver.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths" />
    </provider>
```

3. ### 从一个File得到一个对应的Content URI 
    
    ```java
    private void shareFile() {
         Intent intent = new Intent();
         //处理的客户端activity
         ComponentName componentName = new ComponentName("com.ysx.fileproviderclient",
                 "com.ysx.fileproviderclient.MainActivity");
     intent.setComponent(componentName);
     //从文件得到Uri 
     File file = new File(mContext.getFilesDir() + "/text", "hello.txt");
     Uri data = FileProvider.getUriForFile(mContext, FILE_PROVIDER_AUTHORITIES, file);
     // 给目标应用一个临时授权
     intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
     intent.setData(data);
     //启动目标activity
         startActivity(intent);
     }
    ```

## 其他常见的使用场景：

调用照相机，指定照片存储路径。
调用系统安装器，传递apk文件。
调用照相机：

  * ```java
     /**
  * @param activity 当前activity
       * @param authority FileProvider对应的authority
       * @param file 拍照后照片存储的文件
       * @param requestCode 调用系统相机请求码
         */
     public static void takePicture(Activity activity, String authority, File file, int requestCode) {
         Intent intent = new Intent();
         Uri imageUrii = FileProvider.getUriForFile(activity, authority, file);
         intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);   
         intent.setAction(MediaStore.ACTION_IMAGE_CAPTURE);
         intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
         activity.startActivityForResult(intent, requestCode);
 }
    
```

调用安装器：



  * ```java
     /**
  * 调用系统安装器安装apk *
       * @param context 上下文
       * @param authority FileProvider对应的authority
       * @param file apk文件
         */
      public static void installApk(Context context, String authority, File file) {
         Intent intent = new Intent(Intent.ACTION_VIEW);
         Uri data = FileProvider.getUriForFile(context, authority, file);
         intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
         intent.setDataAndType(data, INSTALL_TYPE);
         context.startActivity(intent);
 }
    
```



## 拿到Content URI之后如何使用？

```java
ParcelFileDescriptor parcelFileDescriptor = getContentResolver().openFileDescriptor(intent.getData(), "r"); 
FileReader reader = new FileReader(parcelFileDescriptor.getFileDescriptor()); 
BufferedReader bufferedReader = new BufferedReader(reader); 
```

## FileProvider本质是个ContentProvider

```java
public class FileProvider extends ContentProvider { 
 
    @Override 
    public boolean onCreate() { 
        return true; 
    } 
 
    @Override 
    public void attachInfo(@NonNull Context context, @NonNull ProviderInfo info) { 
        super.attachInfo(context, info); 
 
        // Sanity check our security 
        if (info.exported) { 
            throw new SecurityException("Provider must not be exported"); 
        } 
        if (!info.grantUriPermissions) { 
            throw new SecurityException("Provider must grant uri permissions"); 
        } 
 
        mStrategy = getPathStrategy(context, info.authority); 
    } 
 
 
    public static Uri getUriForFile(@NonNull Context context, @NonNull String authority, 
            @NonNull File file) { 
        final PathStrategy strategy = getPathStrategy(context, authority); 
        return strategy.getUriForFile(file); 
    } 
 ......
```