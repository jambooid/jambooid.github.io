---
layout: post
title:  "透过FileProvider再看ContentProvider"
date:   2015-10-25 19:16:10
catalog:  true
tags:
    - android组件源码阅读



---

## 透过FileProvider再看ContentProvider

大家应该都熟悉FileProvider吧，但是其诞生的原因，内部怎么实现的，又是怎么转化为文件的，大家有了解多少呢？今天就通过它重新看看ContentProvider这个四大组件之一。

在Android7.0，Android提高了应用的隐私权，限制了在应用间共享文件。如果需要在应用间共享，需要授予要访问的URI临时访问权限。

以下是官方说明：

对于面向 Android 7.0 的应用，Android 框架执行的 StrictMode API 政策禁止在您的应用外部公开 file:// URI。如果一项包含文件 URI 的 intent 离开您的应用，则应用出现故障，并出现 FileUriExposedException 异常。要在应用间共享文件，您应发送一项 content:// URI，并授予 URI 临时访问权限。进行此授权的最简单方式是使用 FileProvider 类。”

**为什么限制在应用间共享文件**

打个比方，应用A有一个文件，绝对路径为file:///storage/emulated/0/Download/photo.jpg

现在应用A想通过其他应用来完成一些需求，比如拍照，就把他的这个文件路径发给了照相应用B，然后应用B照完相就把照片存储到了这个绝对路径。

看起来似乎没有什么问题，但是如果这个应用B是个“坏应用”呢？

- 泄漏了文件路径，也就是应用隐私。

如果这个应用A是“坏应用”呢？

- 自己可以不用申请存储权限，利用应用B就达到了存储文件的这一危险权限。

可以看到，这个之前落伍的方案，从自身到对方，都是不太好的选择。

所以Google就想了一个办法，把对文件的访问限制在应用内部。

- 如果要分享文件路径，不要分享file:// URI这种文件的绝对路径，而是分享content:// URI，这种相对路径，也就是这种格式：content://com.jimu.test.fileprovider/external/photo.jpg
- 然后其他应用可以通过这个绝对路径来向文件所属应用 索要 文件数据，所以文件所属的应用本身必须拥有文件的访问权限。

也就是应用A分享相对路径给应用B，应用B拿着这个相对路径找到应用A，应用A读取文件内容返给应用B。

**配置FileProvider**

从易用性，安全性，完整度等各个方面考虑，Google选择了ContentProvider为这次限制应用分享文件的 解决方案。于是，FileProvider诞生了。

具体做法就是：

```
<!-- 配置FileProvider-->  
<provider     
	android:name="androidx.core.content.FileProvider"
	android:authorities="${applicationId}.provider"    
	android:exported="false"     
	android:grantUriPermissions="true">     
	<meta-data         
		android:name="android.support.FILE_PROVIDER_PATHS"
		android:resource="@xml/provider_paths"/> 
</provider>    

<?xml version="1.0" encoding="utf-8"?> 
<paths xmlns:android="http://schemas.android.com/apk/res/android">     
	<external-path name="external" path="."/> 
</paths> 

//修改文件URL获取方式  
Uri photoURI = FileProvider.getUriForFile(context, context.getApplicationContext().getPackageName() + ".provider", createImageFile()); 
```

这样配置之后，就能生成content:// URI，并且也能通过这个URI来传输文件内容给外部应用。

FileProvider这些配置属性也就是ContentProvider的通用配置：

- android:name，是ContentProvider的类路径。
- android:authorities，是唯一标示，一般为包名+.provider
- android:exported，表示该组件是否能被其他应用使用。
- android:grantUriPermissions，表示是否允许授权文件的临时访问权限。

其中要注意的是android:exported正常应该是true，因为要给外部应用使用。

但是FileProvider这里设置为false，并且必须为false。

这主要为了保护应用隐私，如果设置为true，那么任何一个应用都可以来访问当前应用的FileProvider了，对于应用文件来说不是很可取，所以Android7.0以上会通过其他方式让外部应用安全的访问到这个文件，而不是普通的ContentProvider访问方式，后面会说到。

如果这个属性为true，在Android7.0以下，Android默认是将它当成一个普通的ContentProvider，外部无法通过content:// URI来访问文件。所以一般要判断下系统版本再确定传入的Uri到底是File格式还是content格式。

**FileProvider源码**

接着看看FileProvider的主要源码：

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
 
 
    @Override 
    public Uri insert(@NonNull Uri uri, ContentValues values) { 
        throw new UnsupportedOperationException("No external inserts"); 
    } 
 
 
    @Override 
    public int update(@NonNull Uri uri, ContentValues values, @Nullable String selection, 
            @Nullable String[] selectionArgs) { 
        throw new UnsupportedOperationException("No external updates"); 
    } 
 
 
    @Override 
    public int delete(@NonNull Uri uri, @Nullable String selection, 
            @Nullable String[] selectionArgs) { 
        // ContentProvider has already checked granted permissions 
        final File file = mStrategy.getFileForUri(uri); 
        return file.delete() ? 1 : 0; 
    } 
 
    @Override 
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, 
            @Nullable String[] selectionArgs, 
            @Nullable String sortOrder) { 
        // ContentProvider has already checked granted permissions 
        final File file = mStrategy.getFileForUri(uri); 
 
        if (projection == null) { 
            projection = COLUMNS; 
        } 
 
        String[] cols = new String[projection.length]; 
        Object[] values = new Object[projection.length]; 
        int i = 0; 
        for (String col : projection) { 
            if (OpenableColumns.DISPLAY_NAME.equals(col)) { 
                cols[i] = OpenableColumns.DISPLAY_NAME; 
                values[i++] = file.getName(); 
            } else if (OpenableColumns.SIZE.equals(col)) { 
                cols[i] = OpenableColumns.SIZE; 
                values[i++] = file.length(); 
            } 
        } 
 
        cols = copyOf(cols, i); 
        values = copyOf(values, i); 
 
        final MatrixCursor cursor = new MatrixCursor(cols, 1); 
        cursor.addRow(values); 
        return cursor; 
    } 
 
    @Override 
    public String getType(@NonNull Uri uri) { 
  		final File file = mStrategy.getFileForUri(uri);  
        final int lastDot = file.getName().lastIndexOf('.'); 
        if (lastDot >= 0) { 
            final String extension = file.getName().substring(lastDot + 1); 
            final String mime = MimeTypeMap.getSingleton().getMimeTypeFromExtension(extension); 
            if (mime != null) { 
                return mime; 
            } 
        } 
        return "application/octet-stream"; 
    }  
} 
```

任何一个ContentProvider都需要继承ContentProvider类，然后实现这几个抽象方法：

onCreate，getType，query，insert，delete，update。

（其中每个方法中的Uri参数，就是我们之前通过getUriForFile方法生成的content URI）

我们分三部分说说：

**数据调用方面**

其中，query，insert，delete，update四个方法就是数据的增删查改，也就是进程间通信的相关方法。

其他应用可以通过ContentProvider来调用这几个方法，来完成对本地应用数据的增删查改，从而完成进程间通信的功能。

具体方法就是调用getContentResolver()的相关方法，例如：

```
Cursor cursor = getContentResolver().query(uri, null, null, null, "userid");  
```

再回去看看FileProvider：

- query，查询方法。在该方法中，返回了File的name和length。
- insert，插入方法。没有做任何事。
- delete，删除方法。删除Uri对应的File。
- update，更新方法。没有做任何事。

**MIME类型**

再看getType方法，这个方法主要是返回 Url所代表数据的MIME类型。

一般是使用默认格式：

- 如果是单条记录返回以vnd.android.cursor.item/ 为首的字符串
- 如果是多条记录返回vnd.android.cursor.dir/ 为首的字符串

具体怎么用呢？可以通过Content URI对应的ContentProvider配置的getType来匹配Activity。

有点拗口，比如Activity和ContentProvider这么配置的：

```
<activity   	android:name=".SecondActivity">       
	<intent-filter>           
		<action android:name=""/>           
		<category android:name=""/>           
		<data android:mimeType="type_test"/>      
	</intent-filter>   
</activity>   

@Override public String getType(@NonNull Uri uri) 
{     return "type_test"; }   

intent.setData(mContentRUI);   
startActivity(intent) 
```

这样配置之后，startActivity就会检查Activity的mineType 和 Content URI 对应的ContentProvider的getType是否相同，相同情况下才能正常打开Activity。

**初始化**

最后再看看onCreate方法。

在APP启动流程中，自动执行所有ContentProvider的attachInfo方法，并最后调用到onCreate方法。一般在这个方法中就做一些初始化工作，比如初始化ContentProvider所需要的数据库。

而在FileProvider中，调用了attachInfo方法作为了一个初始化工作的入口，其实和onCreate方法的作用一样，都是App启动的时候会调用的方法。

在这个方法中，也是限制了exported属性必须为false，grantUriPermissions属性必须为true。

```
if (info.exported) {     
	throw new SecurityException("Provider must not be exported"); 
} 
if (!info.grantUriPermissions) {     
	throw new SecurityException("Provider must grant uri permissions"); 
} 
```

这个初始化方法和特性，也是被很多三方库所利用，可以进行静默无感知的初始化工作，而无需单独调用三方库初始化方法。比如Facebook SDK：

```java
<provider 
    android:name="com.facebook.internal.FacebookInitProvider" 
    android:authorities="${applicationId}.FacebookInitProvider" 
    android:exported="false" /> 
	
public final class FacebookInitProvider extends ContentProvider { 
    private static final String TAG = FacebookInitProvider.class.getSimpleName(); 
 
    @Override 
    @SuppressWarnings("deprecation") 
    public boolean onCreate() { 
        try { 
            FacebookSdk.sdkInitialize(getContext()); 
        } catch (Exception ex) { 
            Log.i(TAG, "Failed to auto initialize the Facebook SDK", ex); 
        } 
        return false; 
    } 
 
    //... 
} 
```

这样一写，就无需单独集成FacebookSDK的初始化方法了，实现静默初始化。

而Jetpack中的App Startup也是考虑到这些三方库的需求，对三方库的初始化进行了一个合并，从而优化了多次创建ContentProvider的耗时。

**拿到Content URI 该怎么使用？**

很多人都知道该怎么配置FileProvider让别人（比如照相APP）来获取我们的Content URI，但是你们知道别人拿到Content URI之后又是怎么获取具体的File的呢？

其实仔细找找就能发现，在FileProvider.java中有注释说明：

```
The client app that receives the content URI can open the file and access its contents by calling  {@link android.content.ContentResolver#openFileDescriptor(Uri, String) ContentResolver.openFileDescriptor}   to get a {@link ParcelFileDescriptor} 
```

也就是openFileDescriptor方法，拿到ParcelFileDescriptor类型数据，其实就是一个文件描述符，然后就可以读取文件流了。

```
ParcelFileDescriptor parcelFileDescriptor = getContentResolver().openFileDescriptor(intent.getData(), "r"); 
FileReader reader = new FileReader(parcelFileDescriptor.getFileDescriptor()); 
BufferedReader bufferedReader = new BufferedReader(reader); 
```

**ContentProvider 实际应用**

在平时的工作中，主要有以下以下几种情况和ContentProvider打交道比较多：

- 和系统的一些App通信，比如获取通讯录，调用拍照等。上述的FileProvider也是属于这种情况。
- 与自己的APP有一些交互。比如自家多应用之间，可以通过这个进行一些数据交互。
- 三方库的初始化工作。很多三方库会利用ContentProvider自动初始化这一特性，进行一个静默无感知的初始化工作。

ContentProvider还支持另外一种更直接的数据传输方式，笔者称之为“文件流方式”。因为通过这种方式客户端将得到一个类似文件描述符的对象，然后在其上创建对应的输入或输出流对象。这样，客户端就可通过它们和CP进程交互数据了。

下面来分析这个功能是如何实现的。先向读者介绍客户端使用的函数，即ContentResolver的openAssetFileDescriptor。

```java
public final AssetFileDescriptor openAssetFileDescriptor(Uri uri,
           String mode) throws FileNotFoundException {
    //openAssetFileDescriptor是一个通用函数，它支持三种scheme类型的URI。见
    //下文的解释
   Stringscheme = uri.getScheme();
   if(SCHEME_ANDROID_RESOURCE.equals(scheme)) {
        if(!"r".equals(mode)) {
              throw new FileNotFoundException("Can't write resources: " +uri);
        }
       //创建资源文件输入流
      OpenResourceIdResult r = getResourceId(uri);
        try{
             return r.r.openRawResourceFd(r.id);
         } ......
        }else if (SCHEME_FILE.equals(scheme)) {
           //创建普通文件输入流
           ParcelFileDescriptor pfd = ParcelFileDescriptor.open(
                   new File(uri.getPath()), modeToMode(uri, mode));
           return new AssetFileDescriptor(pfd, 0, -1);
        }else {
           if ("r".equals(mode)) {
                //我们重点分析这个函数，用于读取目标ContentProvider的数据
               return openTypedAssetFileDescriptor(uri, "*/*", null);
           } ......//其他模式的支持，请读者学完本章后自行研究这部分内容
        }
    }
```

如以上代码所述，openAssetFileDescriptor是一个通用函数，它支持三种sheme类型的URI。

- SCHEME_ANDROID_RESOURCE：字符串表达为“android.resource”。通过它可以读取APK包（其实就是一个压缩文件）中封装的资源。假设在应用进程的res/raw目录下存放一个test.ogg文件，最终生成的资源id由R.raw.tet来表达，那么如果应用进程想要读取这个资源，创建的URI就是“android.resource://com.package.name/R.raw.test”。 读者不妨试一试。
- SCHEME_FILE：字符串表达为“file”。通过它可以读取普通文件。
- 除上述两种scheme之外的URI：这种资源背后到底对应的是什么数据需要由目标CP来解释。

接下来分析在第三种sheme类型下调用的openTypedAssetFileDescriptor函数。

1. openTypedAssetFileDescriptor函数分析
   **ContentResolver.java::openTypedAssetFileDescriptor**

```java
public final AssetFileDescriptor openTypedAssetFileDescriptor(Uri uri,
           String mimeType, Bundle opts) throws FileNotFoundException {
    //建立和目标CP进程交互的通道。读者还记得此处provider的真实类型吗？
    //其真实类型是ContentProviderProxy
   IContentProvider provider = acquireProvider(uri);
    try {
           //①调用远端CP的openTypedAssetFile函数，返回值的
          //类型是AssetFileDescriptor。此处传递参数的值为：mimeType="*/*"
          //opts=null
           AssetFileDescriptor fd = provider.openTypedAssetFile(uri,
                                          mimeType, opts);
           ......
           //②创建一个ParcelFileDescriptor类型的变量
           ParcelFileDescriptor pfd = new ParcelFileDescriptorInner(
                   fd.getParcelFileDescriptor(), provider);
           provider = null;
           //③又创建一个AssetFileDescriptor类型的变量作为返回值
           return new AssetFileDescriptor(pfd, fd.getStartOffset(),
                   fd.getDeclaredLength());
        }......
         finally {
           if (provider != null) {
               releaseProvider(provider);
           }
        }
    }
```

在以上代码中，阻碍我们思维的依然是新冒出来的类。先应解决它们，然后再分析代码中的关键调用。
\2. FileDescriptor家族介绍
本节涉及的类家族图谱如图7-7所示。
:-: ![img](http://img.blog.csdn.net/20150803130950821?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
图7-7 FileDescriptor家族图谱

图7-7中的内容比较简单，只需稍作介绍。

- FileDescriptor类是Java的标准类，它是对文件描述符的封装。进程打开的每一个文件都有一个对应的文件描述符。在Native语言开发中，它用一个int型变量来表示。
- 文件描述符作为进程的本地资源，如想越过进程边界将其传递给其他进程，则需借助进程间共享技术。在Android平台上，设计者封装了一个ParcelFileDescriptor类。此类实现了Parcel接口，自然就支持了序列化和反序列化的功能。从图7-7可知，一个ParcelFileDescriptor通过mFileDescritpor指向一个文件描述符。
- AssetFileDescriptor也实现了Parcel接口，其内部通过mFd成员变量指向一个ParcelFileDescriptor对象。从这里可看出，AssetFileDescritpor是对ParcelFileDescriptor类的进一步封装和扩展。实际上，根据SDK文档中对AssetFileDescritpor的描述可知，其作用在于从AssetManager（后续分析资源管理的时候会介绍它）中读取指定的资源数据。

------

**提示**：简单向读者介绍一下与AssetFileDescriptor相关的知识。它用于读取APK包中指定的资源数据。以前面提到的test.ogg为例，如果通过AssetFileDescriptor读取它，那么其mFd成员则指向一个ParcelFileDescriptor对象。且不管这个对象是否跨越了进程边界，它毕竟代表一个文件。假设这个文件是一个APK包，AssetFileDescriptor的mStartOffset变量用于指明test.ogg在这个APK包中的起始位置，比如100字节。而mLength用于指明test.ogg的长度，假设是1000字节。通过上面的介绍可知，该APK文件从100字节到1100字节这一段空间中存储的就是test.ogg的数据。这样，AssetFileDescriptor就能将test.ogg数据从APK包中读取出来了。

**ContentProvider.java::openTypedAssetFile**

```java
public AssetFileDescriptor openTypedAssetFile(Uriuri, String mimeTypeFilter,
                      Bundle opts) throws FileNotFoundException {
    //本例满足下面的if条件
    if("*/*".equals(mimeTypeFilter))
         return openAssetFile(uri, "r");//此函数的代码见下文
 
    StringbaseType = getType(uri);
    if(baseType != null &&
               ClipDescription.compareMimeTypes(baseType, mimeTypeFilter)) {
          return openAssetFile(uri, "r");
    }
   throw newFileNotFoundException("Can't open " +
                uri + " as type " + mimeTypeFilter);
 }
```

**ContentProvider.java::openAssetFile**

```java
public AssetFileDescriptor openAssetFile(Uri uri,String mode)
           throws FileNotFoundException {
   //openFile由子类实现。这里还以MediaProvider为例
    ParcelFileDescriptor fd = openFile(uri, mode);
   //根据openFile返回的fd得到一个AssetFileDescriptor对象
    returnfd != null ? new AssetFileDescriptor(fd, 0, -1) : null;
 }
```

下面分析MediaProvider实现的openFile函数。

1. MediaProvideropenFile分析
   **MediaProvider.java::openFile**

```java
public ParcelFileDescriptor openFile(Uri uri,String mode)
           throws FileNotFoundException {
 
  ParcelFileDescriptor pfd = null;
   //假设URI符合下面的if条件，即客户端想读取的是某音乐文件所属专辑（Album）的信息
   if(URI_MATCHER.match(uri) == AUDIO_ALBUMART_FILE_ID) {
       DatabaseHelper database = getDatabaseForUri(uri);
        ......
       SQLiteDatabase db = database.getReadableDatabase();
       ......
       SQLiteQueryBuilder qb = new SQLiteQueryBuilder();
        //得到客户端指定的代表该音乐文件的_id值
        intsongid = Integer.parseInt(uri.getPathSegments().get(3));
       qb.setTables("audio_meta");
       qb.appendWhere("_id=" + songid);
       Cursor c = qb.query(db,
                   new String [] {
                       MediaStore.Audio.Media.DATA,
                       MediaStore.Audio.Media.ALBUM_ID },
                   null, null, null, null, null);
        if(c.moveToFirst()) {
             String audiopath = c.getString(0);
             //获取该音乐所属的album_id值
             int albumid = c.getInt(1);
             //注意，下面函数中调用的ALBUMART_URI将指向album_art表
             Uri newUri = ContentUris.withAppendedId(ALBUMART_URI, albumid);
             try {
                   //调用ContentProvider实现的openFileHelper函数。注意，pfd的
                   //类型是ParcelFileDescriptor
                   pfd = openFileHelper(newUri, mode);
               } ......
           }
           c.close();
           return pfd;
        }
    ......
}
```

在以上代码中，MediaProvider将首先通过客户端指定的音乐文件的_id去查询它的专辑信息。此处给读者一个示例，如图7-8所示。
:-: ![img](http://img.blog.csdn.net/20150803131007500?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
图7-8 audio_meta内容展示

图7-8中设置的SQL语句是select _id,album_id,_data from audio_meta，得到的结果集包含：第一列音乐文件的_id值，第二列返回音乐文件所属专辑的album_id值，第三列返回对应歌曲的文件存储路径。

以上代码在调用openFileHelper函数前构造了一个新的URI变量，根据代码中的注释可知，它将查询album_art表，不妨再看一个示例，如图7-9所示。
:-: ![img](http://img.blog.csdn.net/20150803131023705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
图7-9 album_art内容展示

在图7-9中，结果集的第一列为专辑艺术家的缩略图文件存储路径，第二列为专辑艺术家album_id值。所以，要打开的文件就是对应album_id的缩略图。再来看openFileHelper的代码。
\2. ContentProvideropenFileHelper函数分析
**ContentProvider.java::openFileHelper**

```java
protected final ParcelFileDescriptoropenFileHelper(Uri uri,
           String mode) throws FileNotFoundException {
  //获取缩略图的文件路径
  Cursor c =query(uri, new String[]{"_data"}, null, null, null);
  int count= (c != null) ? c.getCount() : 0;
  if (count!= 1) {
     ......//一个album_id只能对应一个缩略图文件
  }
 
  c.moveToFirst();
   int i =c.getColumnIndex("_data");
   Stringpath = (i >= 0 ? c.getString(i) : null);
   c.close();
    if (path == null)
           throw new FileNotFoundException("Column _data not found.");
    intmodeBits = ContentResolver.modeToMode(uri, mode);
    //创建ParcelFileDescriptor对象，内部会首先打开一个文件以得到
   //一个FileDescriptor对象，然后再创建一个ParcelFileDescriptor对象，其实
   //就是设置ParcelFileDescriptor中成员变量mFileDescriptor的值
    returnParcelFileDescriptor.open(new File(path), modeBits);
 }
```

至此，服务端已经打开指定文件了。那么，这个服务端的文件描述符是如何传递到客户端的呢？我们单起一节来回答这个问题。

在回答上一节的问题之前，不知读者思考过下面的问题：实现文件描述符跨进程传递的目的是什么？

以上节所示读取音乐专辑的缩略图为例，问题的答案就是，让客户端能够读取专辑的缩略图文件。为什么客户端不先获得对应专辑缩略图的文件存储路径，然后直接打开这个文件，却要如此大费周章呢？原因有二：

- 出于安全的考虑，MediaProvider不希望客户端绕过它去直接读取存储设备上的文件。另外，客户端须额外声明相关的存储设备读写权限，然后才能直接读取其上面的文件。
- 虽然本例针对的是一个实际文件，但是从可扩展性角度看，我们希望客户端使用一个更通用的接口，通过这个接口可读取实际文件的数据，也可读取来自的网络的数据，而作为该接口的使用者无需关心数据到底从何而来。

------

**提示**：实际上还有更多的原因，读者不妨尝试在以上两点原因的基础上拓展思考。

------

1. 序列化ParcelFileDescriptor
   好了，继续讨论本例的情况。现在服务端已经打开了某个缩略图文件，并且获得了一个文件描述符对象FileDescriptor。这个文件是服务端打开的。如何让客户端也打开这个文件呢？各据前文分析，客户端不会也不应该通过文件路径自己去打开这个文件。那该如何处理？

没关系，Binder驱动支持跨进程传递文件描述符。先来看ParcelFileDescriptor的序列化函数writeToParcel，代码如下：
**ParcelFileDescriptor.java::writeToParcel**

```java
public void writeToParcel(Parcel out, int flags) {
   //往Parcel包中直接写入mFileDescriptor指向的FileDescriptor对象
  out.writeFileDescriptor(mFileDescriptor);
   if((flags&PARCELABLE_WRITE_RETURN_VALUE) != 0 && !mClosed) {
        try {
               close();
           }......
        }
    }
```

Parcel的writeFileDescriptor是一个native函数，代码如下：
**android_util_Binder.cpp::android_os_Parcel_writeFileDescriptor**

```java
static voidandroid_os_Parcel_writeFileDescriptor(JNIEnv* env,
                           jobject clazz,jobject object)
{
    Parcel*parcel = parcelForJavaObject(env, clazz);
    if(parcel != NULL) {
        //先调用jniGetFDFromFileDescriptor从Java层FileDescriptor对象中
       //取出对应的文件描述符。在Native层，文件描述符是一个int整型
       //然后调用Native parcel对象的writeDupFileDescriptor函数
       const status_t err =
               parcel->writeDupFileDescriptor(
                                   jniGetFDFromFileDescriptor(env, object));
        if(err != NO_ERROR) {
           signalExceptionForError(env, clazz, err);
        }
    }
}
```

Native Parcel类的writeDupFileDescriptor代码如下：
**Parcel.cpp::writeDupFileDescriptor**

```java
status_t Parcel::writeDupFileDescriptor(int fd)
{
    returnwriteFileDescriptor(dup(fd), true);
}
//直接来看writeFileDescriptor函数
status_t Parcel::writeFileDescriptor(int fd, booltakeOwnership)
{
   flat_binder_object obj;
    obj.type= BINDER_TYPE_FD;
   obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
   obj.handle = fd; //将MediaProvider打开的文件描述符传递给Binder协议
   obj.cookie = (void*) (takeOwnership ? 1 : 0);
    returnwriteObject(obj, true);
}
```

有上边代码可知，ParcelFileDescriptor的序列化过程就是将其内部对应文件的文件描述符取出，并存储到一个由Binder驱动的flat_binder_object对象中。该对象最终会发送给Binder驱动。
\2. 反序列化ParcelFileDescriptor
假设客户端进程收到了来自服务端的回复，客户端要做的就是根据服务端的回复包构造一个新的ParcelFileDescriptor。我们重点关注文件描述符的反序列化，其中调用的函数是Parcel的readFileDescriptor，其代码如下：
**ParcelFileDescriptor.java::readFileDescriptor**

```java
public final ParcelFileDescriptorreadFileDescriptor() {
  //从internalReadFileDescriptor中返回一个FileDescriptor对象
 FileDescriptor fd = internalReadFileDescriptor();
  //构造一个ParcelFileDescriptor对象，该对象对应的文件就是服务端打开的那个缩略图文件
  return fd!= null ? new ParcelFileDescriptor(fd) : null;
```

internalReadFileDescriptor是一个native函数，其实现代码如下：
**android_util_Binder.cpp::android_os_Parcel_readFileDescriptor**

```java
static jobjectandroid_os_Parcel_readFileDescriptor(JNIEnv* env, jobject clazz)
{
    Parcel*parcel = parcelForJavaObject(env, clazz);
    if(parcel != NULL) {
        //调用Parcel的readFileDescriptor得到一个文件描述符
        intfd = parcel->readFileDescriptor();
        if(fd < 0) return NULL;
        fd =dup(fd);//调用dup复制该文件描述符
        if(fd < 0) return NULL;
        //调用jniCreateFileDescriptor以返回一个Java层的FileDescriptor对象
       return jniCreateFileDescriptor(env, fd);
    }
    returnNULL;
}
```

来看Parcel的readFileDescriptor函数，代码如下：
**Parcel.cpp::readFileDescriptor**

```java
int Parcel::readFileDescriptor() const
{
    constflat_binder_object* flat = readObject(true);
    if(flat) {
       switch (flat->type) {
           case BINDER_TYPE_FD:
               //当服务端发送回复包的时候，handle变量指向fd。当客户端接收回复包的时候，
              //又从handle中得到fd。此fd是彼fd吗？
               return flat->handle;
       }       
    }
    returnBAD_TYPE;
}
```

笔者在以上代码中提到了一个较深刻的问题：此fd是彼fd吗？这个问题的真实含义是：

- 服务端打开了一个文件，得到了一个fd。注意，fd是一个整型。在服务端上，这个fd确实对应了一个已经打开的文件。
- 客户端得到的也是一个整型值，它对应的是一个文件吗？

如果说客户端得到一个整型值，就认为它得到了一个文件，这种说法未免有些草率。在以上代码中，我们发现客户端确实根据收到的那个整型值创建了一个FileDescriptor对象。那么，怎样才可知道这个整型值在客户端中一定代表一个文件呢？

这个问题的终极解答在Binder驱动的代码中。来看它的binder_transaction函数。
\3. 文件描述符传递之Binder驱动的处理
**binder.c::binder_transaction**

```java
static void binder_transaction(struct binder_proc*proc,
                      structbinder_thread *thread,
                      structbinder_transaction_data *tr, int reply)
   ......
   switch(fp->type) {
    caseBINDER_TYPE_FD: {
            int target_fd;
            struct file *file;
            if (reply) {
             ......
            //Binder驱动根据服务端返回的fd找到内核中文件的代表file，其数据类型是
            //struct file
            file = fget(fp->handle);
            ......
            //target_proc为客户端进程，task_get_unused_fd_flags函数用于从客户端
           //进程中找一个空闲的整型值，用做客户端进程的文件描述符
            target_fd = task_get_unused_fd_flags(target_proc,
                                           O_CLOEXEC);
            ......
            //将客户端进程的文件描述符和代表文件的file对象绑定
            task_fd_install(target_proc, target_fd, file);
            fp->handle = target_fd;
     }break;
   ......//其他处理
}
```

一切真相大白！原来，Binder驱动代替客户端打开了对应的文件，所以现在可以肯定，客户端收到的整型值确确实实代表一个文件。
\4. 深入讨论
在研究这段代码时，笔者曾经向所在团队同仁问过这样一个问题：在Linux平台上，有什么办法能让两个进程共享同一文件的数据呢？曾得到下面这些回答：

- 两个进程打开同一文件。这种方式前面讨论过了，安全性和可扩展性都比较差，不是我们想要的方式。
- 通过父子进程的亲缘关系，使用文件重定向技术。由于这两个进程关系太亲近，这种实现方式拓展性较差，也不是我们想要的。
- 跳出两个进程打开同一个文件的限制。在两个进程间创建管道，然后由服务端读取文件数据并写入管道，再由客户端进程从管道中获取数据。这种方式和前面介绍的openAssetFileDescriptor有殊途同归之处。

在缺乏类似Binder驱动支持的情况下，要在Linux平台上做到文件描述符的跨进程传递是件比较困难的事。从上面三种回答来看，最具扩展性的是第三种方式，即进程间采用管道作为通信手段。但是对Android平台来说，这种方式的效率显然不如现有的openAssetFileDescriptor的实现。原因在于管道本身的特性。

服务端必须单独启动一个线程来不断地往管道中写数据，即整个数据的流动是由写端驱动的（虽然当管道无空间的时候，如果读端不读取数据，写端也没法再写入数据，但是如果写端不写数据，则读端一定读不到数据。基于这种认识，笔者认为管道中数据流动的驱动力应该在写端）。

Android 3.0以后为ContentProvider提供了管道支持，我们来看相关的函数。
**ContentProvider.java::openPipeHelper**

```java
public <T> ParcelFileDescriptoropenPipeHelper(final Uri uri,
            finalString mimeType, final Bundle opts,
           final T args, final PipeDataWriter<T> func)
           throws FileNotFoundException {
        try {
           //创建管道
           final ParcelFileDescriptor[] fds = ParcelFileDescriptor.
                                              createPipe();
          //构造一个AsyncTask对象
           AsyncTask<Object, Object, Object> task = new
                                    AsyncTask<Object,Object, Object>() {
               @Override
               protected Object doInBackground(Object... params) {
                  //往管道写端写数据，如果没有这个后台线程的写操作，客户端无论如何
                  //也读不到数据的
                   func.writeDataToPipe(fds[1],uri, mimeType, opts, args);
                   try {
                        fds[1].close();
                   } ......
                   return null;
               }
           };
          //AsyncTask.THREAD_POOL_EXECUTOR是一个线程池，task的doInBackground
          //函数将在线程池中的一个线程中运行
           task.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,
                                      (Object[])null);
           return fds[0];//返回读端给客户端
        } ......
 }
```

由以上代码可知，采用管道这种方式的开销确实比客户端直接获取文件描述符的开销大。