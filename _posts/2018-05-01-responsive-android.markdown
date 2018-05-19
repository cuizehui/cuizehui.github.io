---
layout:     post
title:      "Android-H5开发总结"
date:       2018-05-01 20:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Hybrid-App开发
---

# Hybrid混合开发
---

## webview原生开发方式

### js->webview&webview->js

参考以下博客：

[https://blog.csdn.net/carson_ho/article/details/64904691]()
	
[https://juejin.im/post/5a94f9d15188257a63113a74]()

[https://juejin.im/post/5a94fb046fb9a0635865a2d6]()

### webviewClient &webviewChromeClient

webviewClient 是view显示相关api

webviewChrome 是浏览器页面显示相关api 

具体api 可以参考如下文章：
[https://blog.csdn.net/harvic880925/article/details/51523983]()

### 页面回退&页面销毁&页面刷新

```
mWebView.goBack();   
mWebView.goForward()
mWebView.loadurl();
```

### 下拉刷新

SwipeRefreshLayout,并重写webiew滑动事件

----

## Agentweb框架使用

### 简介
agentweb 是对webview进行的又一层封装较为轻量级
所以基本的开发流程大致和webview原理相似
将html5文件方入**asset**文件夹下，访问路径:

```
    final private String CoachFile = "file:///android_asset/teacher/info-teacher.html";

```

#### 运行demo
此demo使用了bintray/Jcenter 这个东西
Jcenter:看这个删除相关部分
[https://blog.csdn.net/u013231041/article/details/70174354]()

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
**注意：** 要调用部分的代码要在script标签定义的fuction中才可以.
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
**注意1：**更改UI方法需要执行在主线程所以需要上面的deliver

**注意2：** AndoidInterface 通常可以进行抽取，因为一些url,uid,token,等native内部资源会被每个页面获取。 而其他特有接口方法应该被抽离出去。



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
#### 刷新|回退|销毁

```
mAgentWeb.getUrlLoader().reload();

if (mAgentWeb.getWebCreator().getWebView().canGoBack()) {
            previousPage();
            mAgentWeb.back();
        } else {
            finish();
 }
 
mAgentWeb.getWebLifeCycle().onDestroy();

```

#### 其他配置

拦截不相关跳转：

``` 
  .interceptUnkownUrl() //拦截找不到相关页面的Url AgentWeb 3.0.0 加入。

```
此段代码添加可能导致自己的文件路径被拦截导致无法跳转。

#### 下拉刷新

使用次第三方库解决下拉刷新smartrefresh

```
    compile 'com.scwang.smartrefresh:SmartRefreshLayout:1.0.3'
```

#### 打开调试功能

```
 AgentWebConfig.debug();

```

## 配置跳转路由表

由于native经常会跳转至不同的html资源文件，所以我们构造了路由表。
本质上是一个hashmap. 此种方式是一个简单解决跳转的方法，如果app内跳转逻辑过多需要组件化开发，可以考虑集成相关路由框架（Activityrouter）框架。

```

public class RouterTable {

    public static final String COACH_SERVICE_FILE = "coachServiceFile";
    public static final String COACH_INFO_FILE = "coachInfoFile";
    ...
    
    public static Map<String, String> routerTable = new HashMap<String, String>() {
        {
            put(COACH_INFO_FILE, "file:///android_asset/teacher/TeacherInfo.html");
            put(COACH_SERVICE_FILE, "file:///android_asset/teacher/TeacherServiceList.html");
   			...
    };

    public static String getTarget(String KEY) {
        return routerTable.get(KEY);
    }
}

```


## 页面间通知刷新

通常更新A界面操作后要通知刷新B界面，采用方案为EventBus 并封装事件

1. 定义事件基类

	 ```
		 public class HTEvent {
		    /**
		     * 事件类型
		     */
		    @IntDef({TYPE_REFRESH})
		    @Retention(RetentionPolicy.SOURCE)
		    private @interface Type {
		    }
		
		    /**
		     * 刷新事件
		     */
		    public static final int TYPE_REFRESH = 0;
		    public @Type
		    int type;
		
		    public HTEvent(@Type int eventType) {
		        this.type = eventType;
		    }
		
		}
	
	```

2. 定义重绘事件，定义对应key
	
	```
		public class HTRefreshEvent extends HTEvent {
			
		/**
		 * 事件类型
		 */
		@IntDef({Type_Refresh_Student_Appointment, Type_Refresh_Coach_Service})
		@Retention(RetentionPolicy.SOURCE)
		private @interface Refresh_Type {
			
		}
			
		/**
		 * 刷新学生Appointment
		 */
		public static final int Type_Refresh_Student_Appointment = 0;
		/**
		 * 刷新教师Service
		 */
		public static final int Type_Refresh_Coach_Service = 1;
			
		public @Refresh_Type
		int refreshType;
			
		public HTRefreshEvent(int eventType, @Refresh_Type int refreshType) {
		    super(eventType);
		    this.refreshType = refreshType;
	    }
		}
	
	```


3. 接受相应刷新事件

	```
	@Subscribe
	    public void onEvent(HTEvent event){
	        if(event.type==HTEvent.TYPE_REFRESH){
	            HTRefreshEvent  refreshEvent= (HTRefreshEvent) event;
	            if(refreshEvent.refreshType==HTRefreshEvent.Type_Refresh_Coach_Service){
	                    mAgentWeb.getWebCreator().getWebView().reload();
	            }
	        }
	    }
	```
	

## H5-Android调试

### 环境搭建方式
1. chrome 插件： chrome://inspect/#devices
2. 允许usb 调试之后点击到对应的html 界面则手机id下方会有相应网址，点击即可进行支持调试的页面进行调试。

***注意**初次使用需要翻墙

### log开启

需要开启如下代码支持调试

```
WebView.setWebContentsDebuggingEnabled(true);　
```
