---
layout:     post
title:      "Android-H5开发总结"
date:       2018-01-16 20:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Hybrid-App开发
---

# Hybrid混合开发
---

## webview原生开发方式

### js->webview|webview->js

参考以下博客：
https://blog.csdn.net/carson_ho/article/details/64904691

### 文件上传（打开本地相册）


## Agentweb框架使用

### 简介
agentweb 是对webview进行的又一层封装较为轻量级
所以基本的开发流程大致和webview原理相似
将html5文件方入asset文件夹下，访问路径为

```
    final private String CoachFile = "file:///android_asset/teacher/info-teacher.html";

```

#### 运行demo
此demo使用了bintray/Jcenter 这个东西
Jcenter:看这个删除相关部分
[https://blog.csdn.net/u013231041/article/details/70174354]
需要在gradle 中将相关代码全部注释掉或者升级对应gradle 版本才能运行

### 使用过程

1. 集成
2. JS-调android
3. Android 调 js
官网给出的代码片段

#### Android 调js (此处待补充)

```
  function callByAndroid(){
      console.log("callByAndroid")
  }
  //此处为agentweb声明js方法
mAgentWeb.getJsAccessEntrace().quickCallJs("callByAndroid");
      
```

#### js->Android

```
//可理解为agentweb注册interface
mAgentWeb.getJsInterfaceHolder().addJavaObject("android",new AndroidInterface(mAgentWeb,this));
window.android.callAndroid();
```

AndroidInterface

```
 public class AndroidInterface {

    private Handler deliver = new Handler(Looper.getMainLooper());
    private AgentWeb agent;
    private Context context;

    public AndroidInterface(AgentWeb agent, Context context) {
        this.agent = agent;
        this.context = context;
    }
	//必须声明此注解

    @JavascriptInterface
    public String getToken(final String msg) {

        String accessToken=Config.getAccessToken(context);
        Log.i("Info", "Thread:" + Thread.currentThread());
        return accessToken;
    }

    @JavascriptInterface
    public int getID(){
        int id=Config.getUid(context);
        Log.d("uid:",""+id);
        return id;
    }
}
```

html调用部分片段

```
	getLocalData:function(){

                  if(window.android!=null&&typeof(window.android)!="undefined"){
                      id=window.android.getID();
                     alert(" : "+id);
                  }else{
                     alert(typeof(window.android));
                  }



      		},
```

#### input标签完成图片和照相并上传

html代码：

```
		<input class="info_image"  type="file"  accept=*/*,"  capture="camera" @change = "uploadImg"/>	
    	
```

webview需要设置WebChromeClient，此框架对WebChromeClient进行封装，以便打开相册和拍照界面。

实现类为：MiddlewareChromeClient | MiddlewareWebViewClient
agentweb进行如下设置：

```
mAgentWeb = AgentWeb.with(this)//
                .useMiddlewareWebClient(getMiddlewareWebClient())
                .useMiddlewareWebChrome(getMiddlewareWebChrome())
                .setWebChromeClient(mWebChromeClient) //WebChromeClient 
```
并要将demo中的实现类clone到自己项目中

在manifest中引入如下依赖：

```
compile 'com.just.agentweb:filechooser:4.0.2'// (可选)
```

完成以上操作即可以调出Android系统的文件相册和拍照功能

input标签拿到的其实是一个file对象可以直接使用，Filereader类。
也可以处理为url 那就要转化成绝对路径。
```
		var files = event.target.files;
		file = files[0];

		var windowURL = window.URL || window.webkitURL;
      dataURL = windowURL.createObjectURL(file);
      //createObjectURL得到的是一个http格式的文件

```

#### 打开调试功能

```
 AgentWebConfig.debug();

```
## H5-Android调试

### 环境搭建方式
chrome 插件： chrome://inspect/#devices
允许usb 调试之后点击到对应的html 界面则手机id下方会有相应网址，点击即可进行支持调试的页面进行调试。
注意初次使用需要翻墙
### log开启

需要开启如下代码支持调试

```
WebView.setWebContentsDebuggingEnabled(true);　
```

## H5界面本地缓存
#### 缓存打包方式

##


