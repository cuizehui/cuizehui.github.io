---
layout:     post
title:      "图片加载方案"
subtitle:   "图片加载，图片缓存处理方案"
date:       2020-03-30 14:50:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android
---


## 简介

本文介绍了大图片加载的思路，介绍了图片缓存的实现思路。

## 图片加载方案

大图片未防止OOM,通常需要将图片裁剪压缩处理

### 图片压缩步骤

 Bitmap尺寸压缩步骤：
    通过采样率加载图片，主要的就是计算出合适的采样率，计算采样率的一般流程：

     1.将BitmapFactory.Options的inJustDecodeBounds参数设置为true并加载图片。

     2.从BitmapFactory.Options中取出原始图片的宽高信息。

     3.根据采样率的规则并结合目标View所需大小计算出采样率inSampleSize。

     4.将BitmapFactory.OPtions的参数设置为false并重新加载图片。是因为它禁止这些方法为bitmap分配内存

原因是inSampleSize只能是2的整数次幂向下取得最大的2的整数次幂，因此可能没有压缩到制定尺寸。压缩后如果图片大小仍大于限制尺寸,则牺牲图片质量进行压缩。

### 实现代码

```java
* 压缩图片，保持图片宽高在768*1024之内（图片宽高不进行拉伸，等比缩放） 大小在120k以下
 *
 * @param srcPath  文件路径
 * @param context  Activity
 * @param savePath 保存路径
 * @param fileName 文件名
 * @param limitSize 限制图片大小
 */
public void compress(String srcPath, Activity context, String savePath, String fileName, int limitSize) throws IOException {
    //设置目标范围为1024*768;
    float hh = 1024;
    float ww = 768;
    //读取bitmap
    BitmapFactory.Options opts = new BitmapFactory.Options();
    //设置只读取大小（防止内存溢出）
    opts.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(srcPath, opts);
    int w = opts.outWidth;
    int h = opts.outHeight;

    int zoom; //大小缩放级别
    Bitmap bitmap = null;
    Bitmap result = null;
    //判断原图宽高,进行粗略缩放防止精确缩放时内存溢出(缩放后的图片)

    //如果宽高都小于目标宽高
    if (w < ww && h < hh) {
        //不进行缩放，直接读取bitmap
        opts.inJustDecodeBounds = false;
        result = BitmapFactory.decodeFile(srcPath, opts);
    } else {
        //图片宽高不在768*1024之内，粗略缩放（缩放结果宽高仍然不在768*1024之内，但是尺寸肯定会小于等于现在尺寸,这样来防止内存溢出）
        //如果宽度缩放比大于高度缩放比，则按照宽度缩放（保持图片宽高比）
        if (w / ww >= h / hh) {
            zoom = (int) (w / ww);
        } else {
            //如果高度缩放比大于宽度缩放比，则按照高度缩放
            zoom = (int) (h / hh);
        }
        //最小为1，不进行缩放
        if (zoom < 1) {
            zoom = 1;
        }
        opts.inSampleSize = zoom;
        //设置读取bitmap
        opts.inJustDecodeBounds = false;
        //读出粗略缩放的bitmap
        bitmap = BitmapFactory.decodeFile(srcPath, opts);
        
         //精确缩放（宽高在768*1024之内，图片等比缩放）
        int w1;    //目标宽度
        int h1;   //目标高度

        //宽度缩放到768,高度等比缩放
        if (bitmap.getWidth() / ww >= bitmap.getHeight() / hh) {
            w1 = (int) ww;
            h1 = (int) (bitmap.getHeight() * (ww / bitmap.getWidth()));
        } else {
            //高度缩放到1024,宽度等比缩放
            h1 = (int) hh;
            w1 = (int) (bitmap.getWidth() * (hh / bitmap.getHeight()));
        }
        //开始精确缩放
        result = zoomImg(bitmap, w1, h1);
        //释放无用的bitmap
        if (bitmap != null && !bitmap.isRecycled()) {
            bitmap.recycle();
            bitmap = null;
        }
        
    }
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
            result.compress(srcPath.endsWith("png") ? Bitmap.CompressFormat.PNG : Bitmap.CompressFormat.JPEG, 100, baos);
            int options = 100;
            while (baos.toByteArray().length > limitSize && options > 0) {
                baos.reset();
                result.compress(srcPath.endsWith("png") ? Bitmap.CompressFormat.PNG : Bitmap.CompressFormat.JPEG, options, baos);
                options -= 10;
            }
            RcsFileUtils.saveFile(savePath, baos.toByteArray());
            result.recycle();        
}

 * 裁剪图片
 *
 * @param bm        位图
 * @param newWidth  新图宽度
 * @param newHeight 新图高度
 * @return
 */
public static Bitmap zoomImg(Bitmap bm, int newWidth, int newHeight) {
    // 获得图片的宽高
    int width = bm.getWidth();
    int height = bm.getHeight();
    // 计算缩放比例
    float scaleWidth = ((float) newWidth) / width;
    float scaleHeight = ((float) newHeight) / height;
    // 取得想要缩放的matrix参数
    Matrix matrix = new Matrix();
    matrix.postScale(scaleWidth, scaleHeight);
    // 得到新的图片
    Bitmap newbm = Bitmap.createBitmap(bm, 0, 0, width, height, matrix, true);
    return newbm;
}
```

## 图片缓存技术

### 实现思路

1.分级缓存使用内存和磁盘分别缓存,如不存在则通过url网络获取
2.内存缓存容器使用LruCache
3.获取时，先判断内存中是否存在，如不存在，则根据路径从磁盘加载值内存。后从内存中加载。

### 代码

```java

public class RcsBitmapCache {

    private final static String TAG = "RcsBitmapCache";
    private final static int BITMAP_LIMIT_SIZE = 100 * 1024;
    private final static int PATH_TAG = 100 << 24;
    private final static int CIRCLE_TAG = 101 << 24;

    private static LruCache<String, Bitmap> sLruCache;
    private static Context sContext;
    private static Executor sExecutor = new ThreadPoolExecutor(5, 10, 10, TimeUnit.SECONDS, new LinkedBlockingDeque<Runnable>(100), new ThreadPoolExecutor.DiscardOldestPolicy());
    private static Map<String, List<WeakReference<ImageView>>> sUrlImageViewsCache = new HashMap<>();

    public static void init(Context context) {
        if (sContext == null) {
            sContext = context;
            int maxMemory = (int) Runtime.getRuntime().maxMemory();
            int lruMemory = maxMemory / 8;
            sLruCache = new LruCache<String, Bitmap>(lruMemory) {
                @Override
                protected int sizeOf(String key, Bitmap bitmap) {
                    // 返回Bitmap对象所占大小，单位：kb
                    return bitmap.getByteCount();
                }

            };
        }
    }

    public static void uninit() {
        sContext = null;
        sLruCache = null;
    }

    public static void getBitmapFromUrl(ImageView imageView, String url, boolean force) {
        if (TextUtils.isEmpty(url)) {
            return;
        }
        String path = genFilePath(url);
        getBitmapFromPath(imageView, path);
        if (force || !new File(path).exists()) {
            imageView.setTag(PATH_TAG, path);
            synchronized (sUrlImageViewsCache) {
                if (sUrlImageViewsCache.keySet().contains(url)) {
                    sUrlImageViewsCache.get(url).add(new WeakReference<>(imageView));
                } else {
                    List<WeakReference<ImageView>> l = new ArrayList<>();
                    l.add(new WeakReference<>(imageView));
                    sUrlImageViewsCache.put(url, l);
                    LoadImageAsyncFromUrl loadImageAsync = new LoadImageAsyncFromUrl(url);
                    loadImageAsync.executeOnExecutor(sExecutor);
                }
            }
        }
    }

    public static void getBitmapFromPath(ImageView imageView, String path) {
        if (!TextUtils.isEmpty(path) && new File(path).exists()) {
            imageView.setTag(PATH_TAG, path);
            Bitmap bitmap = sLruCache.get(path + new File(path).lastModified());
            if (bitmap != null) {
                imageView.setImageBitmap(bitmap);
            } else {
                LoadImageAsyncFromPath loadImageAsync = new LoadImageAsyncFromPath(imageView, path);
                loadImageAsync.executeOnExecutor(sExecutor);
            }
        }
    }

    private static class LoadImageAsyncFromPath extends AsyncTask<Void, Void, Bitmap> {

        private final WeakReference<ImageView> mWeakImageView;
        private final String mPath;

        public LoadImageAsyncFromPath(ImageView imageView, String path) {
            mWeakImageView = new WeakReference<ImageView>(imageView);
            mPath = path;
        }

        @Override
        protected Bitmap doInBackground(Void... params) {
            return compressImage(mPath, BITMAP_LIMIT_SIZE);
        }

        @Override
        protected void onPostExecute(Bitmap bitmap) {
            super.onPostExecute(bitmap);
            ImageView imageView = mWeakImageView.get();
            if (imageView != null) {
                if (bitmap != null) {
                    sLruCache.put(mPath + new File(mPath).lastModified(), bitmap);
                    if (TextUtils.equals((String) imageView.getTag(PATH_TAG), mPath)) {
                        imageView.setImageBitmap(bitmap);
                    }
                }
            }
        }
    }

    private static class LoadImageAsyncFromUrl extends AsyncTask<Void, Void, Bitmap> {

        private final String mUrl;
        private final String mPath;
        private final String mTempPath;

        public LoadImageAsyncFromUrl(String url) {
            mUrl = url;
            mPath = genFilePath(url);
            mTempPath = mPath + ".tmp";
        }

        @Override
        protected Bitmap doInBackground(Void... params) {
            if (HttpUtils.syncHttpDownload(mUrl, mTempPath)) {
                new File(mPath).delete();
                new File(mTempPath).renameTo(new File(mPath));
                return compressImage(mPath, BITMAP_LIMIT_SIZE);
            }
            return null;
        }

        @Override
        protected void onPostExecute(Bitmap bitmap) {
            super.onPostExecute(bitmap);
            synchronized (sUrlImageViewsCache) {
                List<WeakReference<ImageView>> l = sUrlImageViewsCache.get(mUrl);
                if (l != null) {
                    for (WeakReference<ImageView> w : l) {
                        ImageView imageView = w.get();
                        if (imageView != null) {
                            if (bitmap != null) {
                                sLruCache.put(mPath + new File(mPath).lastModified(), bitmap);
                                if (TextUtils.equals((String) imageView.getTag(PATH_TAG), mPath)) {
                                    if ((Boolean) imageView.getTag(CIRCLE_TAG)) {
                                        imageView.setImageDrawable(new CircleDrawable(bitmap));
                                    } else {
                                        imageView.setImageBitmap(bitmap);
                                    }
                                }
                            }
                        }
                    }
                }
                sUrlImageViewsCache.remove(mUrl);
            }
        }
    }


    /**
    *
    * @param srcPath
    * @param limitSize kb
    * @return genrate file
    */
   public static String compressImageToFile(String srcPath, int limitSize) {
       if (srcPath.endsWith("gif")) {
           return srcPath;
       }
       if (!TextUtils.isEmpty(srcPath) && new File(srcPath).exists()) {
           BitmapFactory.Options newOpts = new BitmapFactory.Options();
           newOpts.inJustDecodeBounds = true;
           Bitmap bitmap = BitmapFactory.decodeFile(srcPath, newOpts);
           newOpts.inJustDecodeBounds = false;
           int width = newOpts.outWidth;
           int height = newOpts.outHeight;
           float toSize = 600f;
           int scale = 1;
           if (width > height && width > toSize) {
               scale = (int) (newOpts.outWidth / toSize);
           } else if (width < height && height > toSize) {
               scale = (int) (newOpts.outHeight / toSize);
           }
           if (scale <= 0) {
               scale = 1;
           }
           newOpts.inSampleSize = scale;
           bitmap = BitmapFactory.decodeFile(srcPath, newOpts);
           if (bitmap == null) {
               return null;
           }
           ByteArrayOutputStream baos = new ByteArrayOutputStream();
           bitmap.compress(srcPath.endsWith("png") ? Bitmap.CompressFormat.PNG : Bitmap.CompressFormat.JPEG, 100, baos);
           int options = 100;
           while (baos.toByteArray().length > limitSize && options > 0) {
               baos.reset();
               bitmap.compress(srcPath.endsWith("png") ? Bitmap.CompressFormat.PNG : Bitmap.CompressFormat.JPEG, options, baos);
               options -= 10;
           }
           String destPath = RcsFileUtils.getNotExistFileByTime(RmsDefine.RMS_FILE_PATH, RcsFileUtils.getFileSuffix(srcPath));
           RcsFileUtils.saveFile(destPath, baos.toByteArray());
           bitmap.recycle();
           return destPath;
       }
       return null;
   }
    private static String genFilePath(String url) {
        return RmsDefine.RMS_ICON_PATH + "/" + MD5Utils.md5(url) + "." + RcsFileUtils.getFileSuffix(url);
    }
}

```

## 延伸部分

### 图片的旋转角度

图片如果拍摄时不是正向的可以获取，图片的旋转角度。旋转图片。

```java
 /**
     * 使用路径获取图片旋转角度
     */
    private static int getExifOrientation(String filePath) {
        int degree = 0;
        ExifInterface exif = null;
        try {
            exif = new ExifInterface(filePath);
        } catch (IOException ex) {
            //Log.e("---->", ex.getMessage());
        }
        if (exif != null) {
            int orientation = exif.getAttributeInt(ExifInterface.TAG_ORIENTATION, -1);
            if (orientation != -1) {
                // We only recognize a subset of orientation tag values.
                switch (orientation) {
                    case ExifInterface.ORIENTATION_ROTATE_90:
                        degree = 90;
                        break;
                    case ExifInterface.ORIENTATION_ROTATE_180:
                        degree = 180;
                        break;
                    case ExifInterface.ORIENTATION_ROTATE_270:
                        degree = 270;
                        break;
                    default:
                        break;
                }
            }
        }
        return degree;
    }

```

```java
 /*
     * 旋转图片
     */
    private static Bitmap rotateBitmap(Bitmap src, int degrees) {
        if (degrees != 0 && src != null) {
            Matrix m = new Matrix();
            m.setRotate(degrees, (float) src.getWidth() / 2, (float) src.getHeight() / 2);
            try {
                Bitmap des = Bitmap.createBitmap(src, 0, 0, src.getWidth(), src.getHeight(), m, true);
                if (src != des) {
                    src.recycle();
                    src = des;
                }
            } catch (OutOfMemoryError ex) {

            }
        }
        return src;
    }
```



