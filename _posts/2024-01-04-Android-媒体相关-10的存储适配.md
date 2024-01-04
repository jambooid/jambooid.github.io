---
layout: post
title:  "Android 媒体相关-10+的存储适配"
catalog:  true
tags:
    - android组件源码阅读 


---

> 简介：

# Android 媒体相关-10+的存储适配


 通过本篇文章，你将了解到：

> 1、存储基本知识
>  2、Android 10.0 之前访问方式
>  3、Android 10.0 访问方式变更
>  4、如何不适配Android 10
>  5、MediaStore 基本知识
>  6、通过Uri读取和写入文件
>  7、通过Uri 获取图片和插入相册
>  8、Android 11.0 权限申请
>  9、Android 10/11 存储适配建议

# 1、存储基本知识

先来看看存储区域划分：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0a168a6d4644f86a45de7a60c565ed8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



其中，以下目录无需存储权限即可访问：

> 1、App自身的内部存储
>  2、App自身的自带外部存储-私有目录

剩下的都需要申请存储权限，Android 10.0前后对于存储作用域访问的区别就体现在如何访问剩余这些目录内的文件。

**重点在自带外部存储之共享存储空间和其它目录**

# 2、Android 10.0 之前访问方式

继续细分为Android 6.0 之前和之后。

## Android 6.0 之前访问方式

Android 6.0 之前是无需申请动态权限的，在AndroidManifest.xml 里声明存储权限：

```ini
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

就可以访问共享存储空间、其它目录下的文件了。

## Android 6.0 之后的访问方式

### 动态申请权限

Android 6.0 后需要动态申请权限，除了在AndroidManifest.xml 里声明存储权限外，还需要在代码里动态申请。

```java
//检查权限，并返回需要申请的权限列表
    private List<String> checkPermission(Context context, String[] checkList) {
        List<String> list = new ArrayList<>();
        for (int i = 0; i < checkList.length; i++) {
            if (PackageManager.PERMISSION_GRANTED != ActivityCompat.checkSelfPermission(context, checkList[i])) {
                list.add(checkList[i]);
            }
        }
        return list;
    }

    //申请权限
    private void requestPermission(Activity activity, String requestPermissionList[]) {
        ActivityCompat.requestPermissions(activity, requestPermissionList, 100);
    }

    //用户作出选择后，返回申请的结果
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        if (requestCode == 100) {
            for (int i = 0; i < permissions.length; i++) {
                if (permissions[i].equals(Manifest.permission.WRITE_EXTERNAL_STORAGE)) {
                    if (grantResults[i] == PackageManager.PERMISSION_GRANTED) {
                        Toast.makeText(MainActivity.this, "存储权限申请成功", Toast.LENGTH_SHORT).show();
                    } else {
                        Toast.makeText(MainActivity.this, "存储权限申请失败", Toast.LENGTH_SHORT).show();
                    }
                }
            }
        }
    }

    //测试申请存储权限
    private void testPermission(Activity activity) {
        String[] checkList = new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE, Manifest.permission.READ_EXTERNAL_STORAGE};
        List<String> needRequestList = checkPermission(activity, checkList);
        if (needRequestList.isEmpty()) {
            Toast.makeText(MainActivity.this, "无需申请权限", Toast.LENGTH_SHORT).show();
        } else {
            requestPermission(activity, needRequestList.toArray(new String[needRequestList.size()]));
        }
    }
```

申请权限后，提示用户作出选择：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c49e9833bdcf48c9a8d7ca037738cedd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



### 访问文件

权限申请成功后，即可对自带外部存储之共享存储空间和其它目录进行访问。
 分别以共享存储空间和其它目录为例，阐述访问方式：

#### 访问共享存储空间

共享存储空间分为两类文件：媒体文件和文档/其它文件。

##### 访问媒体文件

目的是拿到媒体文件的路径，有两种方式获取路径：

**1、直接构造路径**
 以图片为例，假设图片存储在/sdcard/Pictures/目录下。

```ini
ini复制代码    private void testShareMedia() {
        //获取目录：/storage/emulated/0/
        File rootFile = Environment.getExternalStorageDirectory();
        String imagePath = rootFile.getAbsolutePath() + File.separator + Environment.DIRECTORY_PICTURES + File.separator + "myPic.png";
        Bitmap bitmap = BitmapFactory.decodeFile(imagePath);
    }
```

如上，myPic.png的路径：/storage/emulated/0/Pictures/myPic.png，拿到路径后就可以解析并获取Bitmap。

**2、通过MediaStore获取路径**
 沿用上篇的demo:

```ini
ini复制代码private void getImagePath(Context context) {
        ContentResolver contentResolver = context.getContentResolver();
        Cursor cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, null);
        while(cursor.moveToNext()) {
            String imagePath = cursor.getString(cursor.getColumnIndex(MediaStore.Images.ImageColumns.DATA));
            Bitmap bitmap = BitmapFactory.decodeFile(imagePath);
            break;
        }
    }
```

同样的，也是拿到图片路径后获取Bitmap。

还有一种不直接通过路径访问的方法：

**3、通过MediaStore获取Uri**

```java
java复制代码    private void getImagePath(Context context) {
        ContentResolver contentResolver = context.getContentResolver();
        Cursor cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, null);
        while(cursor.moveToNext()) {
            //获取唯一的id
            long id = cursor.getLong(cursor.getColumnIndexOrThrow(MediaStore.MediaColumns._ID));
            //通过id构造Uri
            Uri uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id);
            openUri(uri);
            break;
        }
    }
```

与直接拿到路径不同的是，此处拿到的是Uri。图片的信息封装在Uri里，通过Uri构造出InputStream，再进行图片解码拿到Bitmap

##### 访问文档和其它文件

**1、直接构造路径**
 与媒体文件一样，可以直接构造路径访问。

**2、通过SAF访问**
 Storage Access Framework 简称SAF：存储访问框架。相当于系统内置了文件选择器，通过它可以拿到想要访问的文件信息。
 同样的以获取图片为例：

```java
private void startSAF() {
        Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        //选择图片
        intent.setType("image/jpeg");
        startActivityForResult(intent, 100);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == 100) {
            //选中返回的图片封装在uri里
            Uri uri = data.getData();
            openUri(uri);
        }
    }

    private void openUri(Uri uri) {
        try {
            //从uri构造输入流
            InputStream fis = getContentResolver().openInputStream(uri);
            Bitmap bitmap = BitmapFactory.decodeStream(fis);
        } catch (Exception e) {

        }
    }
```

可以看出，通过SAF并不能直接拿到图片的路径，图片的信息封装在Uri里，通过Uri构造出InputStream，再进行图片解码拿到Bitmap。

#### 访问其它目录

有两种方式：
 **1、直接构造路径**
 在/sdcard/目录下直接创建目录：

```ini
ini复制代码    private void testPublicFile() {
        File rootFile = Environment.getExternalStorageDirectory();
        String imagePath = rootFile.getAbsolutePath() + File.separator + "myDir";
        File myDir = new File(imagePath);
        if (!myDir.exists()) {
            myDir.mkdir();
        }
    }
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1196070b321e4ffcbc50be2a3989ca8c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

 可以看出，/sdcard/myDir/目录创建成功。



**2、通过SAF访问**
 与共享存储空间SAF访问方式一致。

## Android 10.0 之前访问方式总结

由上面分析的共享存储空间/其它目录访问方式可知，访问目录/文件可通过如下两个方法：

> 1、通过路径访问。路径可以直接构造也可以通过MediaStore获取。
>  2、通过Uri访问。Uri可以通过MediaStore或者SAF获取。

Android 6.0 以下访问共享存储空间/其它目录步骤：

> 1、AndroidManifest.xml里声明存储权限 2、通过路径或者Uri访问文件

Android 6.0(含)~Android 10.0(不含)访问共享存储空间/其它目录步骤：

> 1、AndroidManifest.xml里声明存储权限
>  2、动态申请存储权限
>  3、通过路径或者Uri访问文件

# 3、Android 10.0 访问方式变更

## 为什么要变更

你可能已经发现了上面访问方式的弊端，比如我们能够直接在/sdcard/目录下创建目录/文件。事实上，很多App就是这么干的，看图说话：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f8a4ff566164a64bccf81b3bce6f43a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

可以看出/sdcard/目录下，如淘宝、qq、qq浏览器、微博、支付宝等都自己建了目录。 这么看来，导致目录结构很乱，而且App卸载后，对应的目录并没有删除，于是就是遗留了很多"垃圾"文件，久而久之不处理，用户的存储空间越来越小。
 总结弊端如下：

> 1、在设置里"Clear storage"或者"Clear cache"并不能删除该目录下的文件
>  2、卸载App也不能删除该目录下的文件
>  3、App可以随意修改其它目录下的文件，如修改别的App创建的文件等，不安全

你也许会问，为什么要在/sdcard/目录下新建自己的目录呢？
 大体有以下两个原因：

> 1、此处新建的目录不会被设置里的App存储用量统计，让用户"看起来"自己的App占用的存储空间很小
>  2、方便操作文件

## 如何变更

面对众多App不讲"码德"随意新建目录/文件的现象，Google在Android 10.0上重拳出击了。

### 引入Scoped Storage

翻译成中文有好几个版本：作用域存储、分区存储、沙盒存储。 具体中文翻译不重要，下面以分区存储指代。 分区存储原理：

> 1、App访问自身内部存储空间、访问外部存储空间-App私有目录不需要任何权限(这个与Android 10.0之前一致)
>  2、外部存储空间-共享存储空间、外部存储空间-其它目录 App无法通过路径直接访问，不能新建、删除、修改目录/文件等
>  3、外部存储空间-共享存储空间、外部存储空间-其它目录 需要通过Uri访问

分区存储的变更在于第二点、第三点。

### 为什么Uri能够访问

先来看为什么通过路径无法直接访问。
 我们知道访问文件最终是通过构造InputStream/OutputStream来实现的，以InputStream为例，看看其构造方法：

```java
java复制代码#FileInputStream.java
    //文件描述符
    private final FileDescriptor fd;
    public FileInputStream(File file) throws FileNotFoundException {
        String name = (file != null ? file.getPath() : null);
        ...
        //传入name，构造FileDescriptor
        //没有权限访问，则此处抛出异常
        fd = IoBridge.open(name, O_RDONLY);
        ...
    }
```

可以看出，要想FileInputStream 能读入文件，核心是需要构造FileDescriptor，而对于Android 10.0，直接通过路径构造FileDescriptor 会抛出异常。
 那么我们自然会想到，有没有通过构造好的FileDescriptor 来生成FileInputStream对象，进而使用read(xx)方法读取数据。
 还真有，请看：通过Uri构造InputStream。

```ini
InputStream fis = getContentResolver().openInputStream(uri);
```

进入看其源码：

```java
ContentResolver.java
    public final @Nullable
    InputStream openInputStream(@NonNull Uri uri)
            throws FileNotFoundException {
        ...
        if (SCHEME_ANDROID_RESOURCE.equals(scheme)) {
            ...
        } else if (SCHEME_FILE.equals(scheme)) {
            ...
        } else {
            //通过Uri构造fd是被允许的
            AssetFileDescriptor fd = openAssetFileDescriptor(uri, "r", null);
            try {
                //反过来创建InputStream
                return fd != null ? fd.createInputStream() : null;
            } catch (IOException e) {
                throw new FileNotFoundException("Unable to create stream");
            }
        }
    }
```

AssetFileDescriptor 持有ParcelFileDescriptor 引用，而ParcelFileDescriptor 持有FileDescriptor 引用。
 同理也适用于FileOutputStream。因此，通过Uri能够访问文件。

# 4、如何不适配Android 10.0

> 1、Android 一般升级功能的时候都会配合targetSdkVersion使用。只要targetSdkVersion<=28，分区存储功能就不会开启。

> 2 在AndroidManifest.xml 里application标签下添加： android:requestLegacyExternalStorage="true" 可禁用分区存储

Android11已经强制要求开启SS

# 5、MediaStore 基本知识

上篇已经分析得出结论，Android 10.0 存储访问方式变更地方在于：

> 外部存储-共享存储空间
>
> 外部存储-其它目录

以上两个地方不能通过路径直接访问文件，而是需要通过Uri访问。

## 共享存储空间

共享存储空间存放的是图片、视频、音频等文件，这些资源是公用的，所有App都能够访问它们。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57355524844849238928eca5222e5cb5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



系统里有external.db数据库，该数据库里有files表，该表里存放着共享文件的诸多信息，如图片有宽高，经纬度、存放路径等，视频宽高、时长、存放路径等。而文件真正存放的地方在于共享存储空间。

**1、保存图片到相册**
 当App1保存图片到相册时，简单流程如下：

> 1、将路径信息写入数据库里，并获取Uri
>  2、通过Uri构造输出流
>  3、将该图片保存在/sdcard/Pictures/目录下

**2、从相册获取图片**
 当App2从相册获取图片时，简单流程如下：

> 1、先查询数据库，找到对应的图片Cursor
>  2、从Cursor里构造Uri
>  3、从Uri构造输入流读取图片

以上以图片为例简单分析了共享存储空间文件的写入与读取，实际上对于视频、音频步骤亦是如此。

## MediaStore作用

共享存储空间里存放着图片、视频、音频、下载的文件，App获取或者插入文件的时候怎么区分这些类型呢？
 这个时候就需要MediaStore，来看看MediaStore.java

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c076cf9bcd944d8b44cb69c505602aa~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 


 可以看出其内部有Audio、Images等内部类，这些内部类里记录着files表的各个字段名，通过构造这些参数就可以插入相应的字段值以及获取对应的字段值。 MediaStore 实际上就是相当于给各个字段起了别名，我们编码的时候更容易记住与使用：

```arduino
//列举一些字段：
//图片类型
MediaStore.Images.Media.MIME_TYPE
//音频时长
MediaStore.Audio.Media.DURATION
//视频时长
MediaStore.Video.Media.DURATION
//等等，还有很多
```

### MediaStore和Uri联系

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/875ffc81eeb0487bafdebc649c571eb8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)


 比如想要查询共享存储空间里的图片文件：

```sql
sql复制代码Cursor cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, null);
```

MediaStore.Images.Media.EXTERNAL_CONTENT_URI 意思是指定查询文件的类型是图片，并构造成Uri对象，Uri实现了Parcelable，能够在进程间传递。
 接收方(另一个进程收到后)，匹配Uri，解析出对应的字段，进行具体的操作。
 当然，MediaStore是系统提供的方便操作共享存储空间的类，若是自己写ContentProvider，则也可以自定义类似MediaStore的类用来标记自己的数据库表的字段。

# 6、通过Uri读取和写入文件

既然不能通过路径直接访问文件，那么来看看如何通过Uri访问文件。在上篇文章里提到过：**Uri可以通过MediaStore或者SAF获取。**(此处需要注意的是：虽然也可以通过文件路径直接构造Uri，但是此种方式构造的Uri是没有权限访问文件的)
 先来看看通过SAF获取Uri。

## 从Uri读取文件

现在/sdcard/目录下存在一个文件名为：mytest.txt。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcc4cbe9763f4cb781a818876be53b70~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)  



该文件内容是：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e09f6d524b1e46b2b58f0429a7f5d3af~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)  



传统的直接读取mytest.txt方法：

```java
    //从文件读取
    private void readFile(String filePath) {
        if (TextUtils.isEmpty(filePath))
            return;

        try {
            File file = new File(filePath);
            FileInputStream fileInputStream = new FileInputStream(file);
            BufferedInputStream bis = new BufferedInputStream(fileInputStream);
            byte[] readContent = new byte[1024];
            int readLen = 0;
            while (readLen != -1) {
                readLen = bis.read(readContent, 0, readContent.length);
                if (readLen > 0) {
                    String content = new String(readContent);
                    Log.d("test", "read content:" + content.substring(0, readLen));
                }
            }
            fileInputStream.close();
        } catch (Exception e) {

        }
    }
```

开启分区存储功能后，这种方法是不可取的，会报权限错误。
 而mytest.txt不属于共享存储空间的文件，是属于其它目录的，因此不能通过MediaStore获取，只能通过SAF获取，如下：

```java
private void startSAF() {
        Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        //指定选择文本类型的文件
        intent.setType("text/plain");
        startActivityForResult(intent, 100);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == 100) {
            //选中返回的文件信息封装在Uri里
            Uri uri = data.getData();
            openUriForRead(uri);
        }
    }
```

拿到Uri后，用来构造输入流读取文件。

```java
private void openUriForRead(Uri uri) {
        if (uri == null)
            return;
        try {
            //获取输入流
            InputStream inputStream = getContentResolver().openInputStream(uri);
            byte[] readContent = new byte[1024];
            int len = 0;
            do {
                //读文件
                len = inputStream.read(readContent);
                if (len != -1) {
                    Log.d("test", "read content:" + new String(readContent).substring(0, len));
                }
            } while (len != -1);
            inputStream.close();
        } catch (Exception e) {
            Log.d("test", e.getLocalizedMessage());
        }
    }
```

最终输出：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe89b50a03444616a393e3bc80594219~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



由此可以看出，mytest.txt属于"其它目录"下的文件，因此需要通过SAF访问，SAF返回Uri，通过Uri构造InputStream即可读取文件。

## 从Uri写入文件

继续来看看写的过程，现在需要往mytest.txt写入内容。
 同样的，还是需要通过SAF拿到Uri，拿到Uri后构造输出流：

```scss
private void openUriForWrite(Uri uri) {
        if (uri == null) {
            return;
        }

        try {
            //从uri构造输出流
            OutputStream outputStream = getContentResolver().openOutputStream(uri);
            //待写入的内容
            String content = "hello world I'm from SAF\n";
            //写入文件
            outputStream.write(content.getBytes());
            outputStream.flush();
            outputStream.close();
        } catch (Exception e) {
            Log.d("test", e.getLocalizedMessage());
        }
    }
```

最后来看看文件是否写入成功，通过SAF再次读取mytest.txt，发现正好是之前写入的内容，说明写入成功。

# 7、通过Uri 获取图片和插入相册

上面列举出了其它目录下文件的读写，方法是通过SAF拿到Uri。
 SAF好处是：

> 系统提供了文件选择器，调用者只需要指定想要读写的文件类型，比如文本类型、图片类型、视频类型等，选择器就会过滤出相应文件以供选择。接入方便，选择简单。

想想另一种场景：

> 想要自己实现相册选择器，那么就需要获得共享存储空间下的文件信息。此种场景下使用SAF是无法做到的。

因此问题的关键是：**如何批量获得共享存储空间下图片/视频的信息？**
 答案是：ContentResolver+ContentProvider+MediaStore(ContentProvider对于调用者是透明的)。
 以图片为例，分析插入与查询方式。
 来看看图片的插入过程：

```java
//fileName为需要保存到相册的图片名
    private void insert2Album(InputStream inputStream, String fileName) {
        if (inputStream == null)
            return;
        ContentValues contentValues = new ContentValues();
        contentValues.put(MediaStore.Images.ImageColumns.DISPLAY_NAME, fileName);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            //RELATIVE_PATH 字段表示相对路径-------->(1)
            contentValues.put(MediaStore.Images.ImageColumns.RELATIVE_PATH, Environment.DIRECTORY_PICTURES);
        } else {
            String dstPath = Environment.getExternalStorageDirectory() + File.separator + Environment.DIRECTORY_PICTURES
                    + File.separator + fileName;
            //DATA字段在Android 10.0 之后已经废弃
            contentValues.put(MediaStore.Images.ImageColumns.DATA, dstPath);
        }
        //插入相册------->(2)
        Uri uri = getContentResolver().insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues);
        //写入文件------->(3)
        write2File(uri, inputStream);
    }
```

重点说明三个点：
 **(1)**
 Android 10.0之前，MediaStore.Images.ImageColumns.DATA 字段记录的是图片的绝对路径，而Android 10.0(含)之后，DATA 被废弃，取而代之的是使用MediaStore.Images.ImageColumns.RELATIVE_PATH，表示相对路径。比如指定RELATIVE_PATH为Environment.DIRECTORY_PICTURES，表示之后的图片将会放到Environment.DIRECTORY_PICTURES目录下。

**(2)**
 调用ContentResolver里的方法插入相册。
 MediaStore.Images.Media.EXTERNAL_CONTENT_URI 指的是插入图片表。
 ContentValues 以Map的形式记录了待写入的字段值。
 插入后返回Uri。

**(3)**
 以上两步仅仅只是往数据库里增加一条记录，该记录指向的新文件是空的，需要将图片写入到新文件。
 而新文件位于/sdcard/Pictures/目录下，该目录是不能直接通过路径访问的，因此需要通过第二步返回的Uri进行访问。

```java
//uri 关联着待写入的文件
    //inputStream 表示原始的文件流
    private void write2File(Uri uri, InputStream inputStream) {
        if (uri == null || inputStream == null)
            return;

        try {
            //从Uri构造输出流
            OutputStream outputStream = getContentResolver().openOutputStream(uri);

            byte[] in = new byte[1024];
            int len = 0;

            do {
                //从输入流里读取数据
                len = inputStream.read(in);
                if (len != -1) {
                    outputStream.write(in, 0, len);
                    outputStream.flush();
                }
            } while (len != -1);

            inputStream.close();
            outputStream.close();

        } catch (Exception e) {
            Log.d("test", e.getLocalizedMessage());
        }
    }
```

可以看出，目标文件关联的Uri有了，还需要原始的输入文件。

测试上述的插入方法：

```ini
private void testInsert() {

        String picName = "mypic.jpg";
        try {
            File externalFilesDir = getExternalFilesDir(null);
            File file = new File(externalFilesDir, picName);
            FileInputStream fis = new FileInputStream(file);
            insert2Album(fis, picName);
        } catch (Exception e) {
            Log.d("test", e.getLocalizedMessage());
        }
    }
```

其中，原始文件(图片)存放于自带外部存储-App私有目录，如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/448f70b55e2440eca22d612b1103ba10~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



需要注意的是：

> 1、读取原始文件需要权限，上述例子里的原始文件存放在自带外部存储-App私有目录，因此本App可以使用路径直接读取
>  2、对于其他目录则依然需要构造Uri读取，如通过SAF获取Uri

## 获取图片

同样的，想要从系统相册中获取图片，也需要通过Uri访问。

```java
private void queryImageFromAlbum() {
        Cursor cursor = getContentResolver().query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null,
                null, null, null);

        if (cursor != null) {
            while (cursor.moveToNext()) {
                //获取唯一的id
                long id = cursor.getLong(cursor.getColumnIndexOrThrow(MediaStore.MediaColumns._ID));
                //通过id构造Uri
                Uri uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id);
                //解析uri
                decodeUriForBitmap(uri);
            }
        }
    }

    private void decodeUriForBitmap(Uri uri) {
        if (uri == null)
            return;

        try {
            //构造输入流
            InputStream inputStream = getContentResolver().openInputStream(uri);
            //解析Bitmap
            Bitmap bitmap = BitmapFactory.decodeStream(inputStream);
            if (bitmap != null)
                Log.d("test", "bitmap width-width:" + bitmap.getWidth() + "-" + bitmap.getHeight());
        } catch (Exception e) {
            Log.d("test", e.getLocalizedMessage());
        }
    }
```

与插入相册过程类似，同样需要拿到Uri，再构造输入流，从输入流读取文件(图片内容)。

以上，通过Uri 获取图片和插入相册分析完毕，共享存储空间的其他文件类型如视频、音频、下载文件也是同样的流程。
 需要说明的是上述的ContentResolver .insert(xx)/ContentResolver.query(xx) 的参数取值还可以更丰富，但不是本篇重点，因此忽略了，实际使用过程中具体情况具体分析。

# 8、Android 11.0 权限申请

通过Uri访问文件似乎已经满足了Android 10.0适配要求，但是仔细想想还是有不足之处：

> 1、共享存储空间只能通过MediaStore访问，以前流行的访问方式是直接通过路径访问。比如自己做的相册管理器，先遍历相册拿到图片/视频的路径，然后再解析成Bitmap展示，现在需要先拿到Uri，再解析成Bitmap，多少有些不方便。此外，也许你依赖的第三方库是直接通过路径访问文件的，而三方库又没有及时更新适配分区存储，可能就会导致用不了相应的功能。
>  2、SAF虽然能够访问其它目录的文件，但是每次都需要跳转到新的页面去选择，当想要批量展示文件的时候，比如自己做的文件管理器，就需要列出当前目录下有哪些目录/文件，这个时候需要有权限遍历/sdcard/目录。显然，SAF并不能胜任此工作。

Android 11.0考虑到上面的问题，因此做了新的优化。

## 共享存储空间-媒体文件访问变更

媒体文件可以通过路径直接访问：

```java
private void getImagePath(Context context) {
        ContentResolver contentResolver = context.getContentResolver();
        Cursor cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, null);
        while (cursor.moveToNext()) {
            try {
                //取出路径
                String path = cursor.getString(cursor.getColumnIndex(MediaStore.Images.ImageColumns.DATA));
                Bitmap bitmap = BitmapFactory.decodeFile(path);
            } catch (Exception e) {
                Log.d("test", e.getLocalizedMessage());
            }
            break;
        }
    }
```

可以看出，之前在Android 10.0上被禁用的访问方式，在Android 11.0上又被允许了，这就解决了上面的第一个问题。
 *需要注意的是：此种方式只允许读文件，写文件依然不行*

Google 官方指导意见是：

> 虽然可以通过路径直接访问媒体文件，但是这些操作最终是被重定向到MediaStore API的，重定向过程可能会损耗一些性能，并且直接通过路径访问不一定比MediaStore API 访问快。 总之建议非必要的话不要直接使用路径访问。

## 访问所有文件

假若App开启了分区存储功能，当App运行在Android 10.0的设备上时，是没法遍历/sdcard/目录的。  而在Android 11.0上运行时是可以遍历的，需要进行如下几个步骤。

### 1、声明管理权限

在AndroidManifest.xml添加权限声明

```ini
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
```

### 2、动态申请所有文件访问权限

```scss
scss复制代码    private void testAllFiles() {
        //运行设备>=Android 11.0
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            //检查是否已经有权限
            if (!Environment.isExternalStorageManager()) {
                //跳转新页面申请权限
                startActivityForResult(new Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION), 101);
            }
        }
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        //申请权限结果
        if (requestCode == 101) {
            if (Environment.isExternalStorageManager()) {
                Toast.makeText(MainActivity.this, "访问所有文件权限申请成功", Toast.LENGTH_SHORT).show();

                 //遍历目录
                showAllFiles();
            }
        }
    }
```

此处申请权限不是以对话框的形式提示用户，而是跳转到新的页面，说明该权限的管理更严格。

### 3、遍历目录、读写文件

拥有权限后，就可以进行相应的操作了。

```ini
private void showAllFiles() {
        File file = Environment.getExternalStorageDirectory();
        File[] list = file.listFiles();
        for (int i = 0; i < list.length; i++) {
            String name = list[i].getName();
            Log.d("test", "fileName:" + name);
        }
    }
```

文件管理器效果图类似如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6ce7a2fc89a4e01aa95270d78bf9abd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



当然读写文件也不在话下了，比如往/sdcard/目录下写入文件：

```ini
ini复制代码    private void testPublicFile() {
        File rootFile = Environment.getExternalStorageDirectory();
        try {
            File file = new File(rootFile, "mytest.txt");
            FileOutputStream fos = new FileOutputStream(file);
            String content = "hello world\n";
            fos.write(content.getBytes());
            fos.flush();
            fos.close();
        } catch (Exception e) {
            Log.d("test", e.getLocalizedMessage());
        }
    }
```

ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION 这个权限的名字看起来很唬人，感觉就像是能够操作所有文件的样子，这不就是打破了分区存储的规则了吗？其实不然：

> 即使拥有了该权限，依然不能访问内部存储和外部存储-App私有目录

需要说明的是：

> 1、Environment.isExternalStorageManager()、Build.VERSION_CODES.R 等需要编译版本>=30才能编译通过。
>  2、Google 提示当使用MANAGE_EXTERNAL_STORAGE 申请权限时，并且targetSdkVersion>=30，此种情况下App被禁止上架Google Play的，限制时间最早到2021年。因此，在此时间之前若是申请了MANAGE_EXTERNAL_STORAGE权限，最好不要升级targetSdkVersion到30以上。

# 9、Android 10/11 存储适配建议

好了，通过分析Android 10/11存储适配方式，了解到了不同的系统需要如何进行适配，此时就需要一个统一的适配方案了。

## 适配核心

分区存储是核心，App自身产生的文件应该存放在自己的目录下：

> /sdcard/Android/data/packagename/ 和/data/data/packagename/

这两个目录本App无需申请访问权限即可申请，其它App无法访问本App的目录。

## 适配共享存储

共享存储空间里的文件需要通过Uri构造输入输出流访问，Uri获取方式有两种：MediaStore和SAF。

## 适配其它目录

在Android 11上需要申请访问所有文件的权限。

## 具体做法

### 第一步

在AndroidManifest.xml里添加如下字段： 权限声明：

```ini
ini复制代码    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
```

在标签下添加如下字段：

```ini
ini复制代码android:requestLegacyExternalStorage="true"
```

### 第二步

如果需要访问共享存储空间，则判断运行设备版本是否大于等于Android6.0，若是则需要申请WRITE_EXTERNAL_STORAGE 权限。拿到权限后，通过Uri访问共享存储空间里的文件。
 如果需要访问其它目录，则通过SAF访问

### 第三步

如果想要做文件管理器、病毒扫描管理器等功能。则判断运行设备版本是否大于等于Android 6.0，若是先需要申请普通的存储权。若运行设备版本为Android 10.0，则可以直接通过路径访问/sdcard/目录下文件(因为禁用了分区存储)；若运行设备版本为Android 11.0，则需要申请MANAGE_EXTERNAL_STORAGE 权限。

以上是Android 存储权限适配的全部内容。