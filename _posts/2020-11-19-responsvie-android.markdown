---
layout:     post
title:      "沙盒模式文件适配"
subtitle:   "双进程沙盒模式文件业务适配"
date:       2020-11-19 20:50:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android-FrameWork
---

# AndroidQ 沙盒模式文件适配

## 需求

Android系统升级至AndroidR强制开启沙盒模式，无法读取私有目录，无法读取操作sdcard下目录文件。 

根据RCS业务需要,部分厂商采取双进程方式。而文件上传下载续传续载属于业务进程,文件展示在短信进程。故需要处理双进程文件读取问题。


## 沙盒模式

1. 沙盒内创建的文件夹及文件会随着应用的卸载一并删除,目录为sdcard/Android/data/packageName/file
2. 限制了APP向SDcard中读写文件
3. MediaStore媒体公共访问目录,在沙盒模式下只能访问音频照片视频，不随应用卸载而删除。通过数据库uri查询获取。（有文件类型限制）
   ```
   MediaStore是外部存储空间的公共媒体集合，存放的都是多媒体文件
   照片：存储在 MediaStore.Images 
   视频：存储在 MediaStore.Video 
   音乐文件：存储在 MediaStore.Audio 
   下载文件：存储在 MediaStore.Downloads 在API >= 29后加入
   所有文件：存储在 MediaStore.Files 
   ```
4. FileProviderXML构造文件ContentUri需要授权且超过有效期无法使用    

## 解决思路和方案

1. 业务进程不需要永久持有文件可在完成业务操作后删除文件
2. ContentProvider接口提供进程间通信方式获取文件


## 流程图

![message](/img/in-post/sandboxFileUpload.png)

![message](/img/in-post/sandboxFileDownload.png)



## 关键代码

### 进程1提供fd

```java

public class RmsProvider extends ContentProvider {

    private final static String TAG = "RmsProvider";
    private static final Uri NOTIFICATION_URI = Uri.parse("content://" + RmsDefine.RMS_AUTHORITY);
    private RcsDatabaseHelper mDatabaseHelper;

    @Override
    public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
        switch (sURLMatcher.match(uri)) {
            case RMS_LOG_FILE:
                //删除文件
                String filename = uri.getPathSegments().get(1);
                if (TextUtils.isEmpty(filename)) {
                    return null;
                }
                File file = new File(RmsDefine.RMS_FILE_PATH, filename);
                if (file.exists()) {
                    return ParcelFileDescriptor.open(file,
                            ParcelFileDescriptor.MODE_READ_ONLY);
                } else {
                    return null;
                }
            default:
                Log.e("RmsProvider","Unknown Uri");
                break;
        }
        return null;
    }

    @Override
    public int delete(Uri url, String where, String[] whereArgs) {
        int count = 0;
        int match = sURLMatcher.match(url);
        SQLiteDatabase db = mDatabaseHelper.getWritableDatabase();

        switch (match) {
            case RMS_LOG_FILE:
                //删除文件
                String filename = url.getPathSegments().get(1);
                if (TextUtils.isEmpty(filename)) {
                    return 0;
                }
                File file = new File(RmsDefine.RMS_FILE_PATH, filename);
                if (file.exists() && file.isFile()) {
                    if (file.delete()) {
                        return 1;
                    } else {
                        return 0;
                    }
                } else {
                    return 0;
                }
            default:
                throw new IllegalArgumentException("Unknown URL");
        }
        return count;
    }


    @Override
    public boolean onCreate() {
        mDatabaseHelper = RcsDatabaseHelper.getInstance(getContext());
        return true;
    }

    private static final int RMS_LOG = 0;
    private static final int RMS_LOG_ID = 1;
    private static final int RMS_LOG_FILE = 2;

    private static final UriMatcher sURLMatcher = new UriMatcher(
            UriMatcher.NO_MATCH);

    static {
        sURLMatcher.addURI(RmsDefine.RMS_AUTHORITY, "rms_log", RMS_LOG);
        sURLMatcher.addURI(RmsDefine.RMS_AUTHORITY, "rms_log/#", RMS_LOG_ID);
        sURLMatcher.addURI(RmsDefine.RMS_AUTHORITY, "rms_log_file/*", RMS_LOG_FILE);
    }
}

```

### 构建uri

```java
    public static final Uri CONTENT_URI_PATH = Uri.parse("content://rms/rms_log_file");
    RmsDefine.Rms.CONTENT_URI_PATH.buildUpon().appendPath(MtcImFthttp.Mtc_ImFtHttpGetName(dwFtHttpId)).build().toString());
```

### 进程2通过fd获取文件

```java
     FileDescriptor fileDescriptor = RcsMmsInitHelper.getContext().getContentResolver().openFileDescriptor(uri, "r").getFileDescriptor();
          
          
          /**
     * 复制文件到目标路径
     * @param src 原文件fd
     * @param dest 目标路径
     * @return
     */
    public static boolean copyFile(FileDescriptor src, String dest) {
        boolean result = false;
        FileChannel srcChannel = null;
        FileChannel dstChannel = null;
        if ((src == null) || TextUtils.isEmpty(dest)) {
            return false;
        }
        File file = new File(dest);
        try {
            if (!file.exists()) {
                file.createNewFile();
            }
            srcChannel = new FileInputStream(src).getChannel();
            dstChannel = new FileOutputStream(file).getChannel();
            srcChannel.transferTo(0, srcChannel.size(), dstChannel);
            result = true;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (srcChannel != null) {
                    srcChannel.close();
                }
                if (dstChannel != null) {
                    dstChannel.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return result;
    }
       
                    
```

### 线程池拷贝文件

```java
    private static final ExecutorService exec = new ThreadPoolExecutor(5, 10, 10,TimeUnit.SECONDS, new LinkedBlockingDeque<Runnable>(100), new ThreadPoolExecutor.DiscardOldestPolicy());

    
    exec.submit(new Runnable() {
                @Override
                public void run() {
                    Uri uri = Uri.parse(jsonObj.optString(RcsJsonParamConstants.RCS_JSON_FILE_URI));
                    try {
                        FileDescriptor fileDescriptor = RcsMmsInitHelper.getContext().getContentResolver().openFileDescriptor(uri, "r").getFileDescriptor();
                        RcsFileUtils.copyFile(fileDescriptor, jsonObj.optString(RcsJsonParamConstants.RCS_JSON_FILE_PATH));
                        Intent intent = new Intent(RcsJsonParamConstants.RCS_ACTION_IM_NOTIFY);
                        jsonObj.remove(RcsJsonParamConstants.RCS_JSON_FILE_URI);
                        intent.putExtra(RcsJsonParamConstants.RCS_JSON_KEY, jsonObj.toString());
                        intent.setClass(RcsMmsInitHelper.getContext(), RcsImReceiverService.class);
                        RcsMmsInitHelper.getContext().getContentResolver().delete(uri, null, null);
                        RcsImReceiverService.enqueueWork(RcsMmsInitHelper.getContext(), intent);
                    } catch (FileNotFoundException e) {
                        e.printStackTrace();
                    }
                }
            });
           
```

**ContentProviderURi找不到问题**

targetAPI升级至30无法读取其他进程ContentProvider需要在AndroidManifest增加以下代码

```xml
    <queries>
        <package android:name="包名" />
    </queries>

```

## 可优化部分

1. 相同文件需要反复复制删除



