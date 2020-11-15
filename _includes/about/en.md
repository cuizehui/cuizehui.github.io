# 个人信息

 - 崔泽辉      男    1994 
 - **本科 | 华东交通大学（一本）** | 物联网工程系 
 - 工作年限：   *3年*
 - 期望职位： Android开发工程师
 - 个人博客： [https://cuizehui.github.io/](https://cuizehui.github.io/) 
 - 手机：18668579239    
 - Email： 454776824@qq.com   
 - WeChat：czh34351

---

# 个人技能简介

- 有音视频SDK开发经验（1年），RCS-SDK开发经验
- 有Android framework 开发经验（2年）原生短信（MMS & Messaging）源码开发的经验
- 熟悉Git分支操作,熟练掌握AndroidStudio使用,熟悉Gerrit+Jenkins平台,熟悉Gradle基本操作
- 熟练掌握Mac/Ubuntu/Windows系统下的开发操作
- 有良好的代码规范,命名规范，接口文档规范，并能够熟练使用MarkDown
- Java基础基础良好,掌握正则,Socket,反射等技巧。掌握常用设计模式：单例、享元、工厂，原型，动态代理等,能设计并使用mvc/mvp框架进行开发
- Android基础良好,能够完成大多数界面开发,并快速学习UI控件使用,对Binder进程间通信有过开发经验
- 能够编写简单Python脚本，对JavaSDK接口自动化测试有搭建经验

# 工作经历

## **宁波菊风系统软件有限公司**（ 2018年4月 ~ 至今 ）

#### 融合通信短信开发 （2019年1月-至今）

 业务描述：
 
 按照中国移动规范文档，基于短信源码，**将短信开发为增强信息**并集成Maap业务（类似微信公众账号）,**使短信具有群组聊天、发送图片、语音、地理位置等功能**。
    
   - 完成的模块
   
     商家发现，搜索功能、文件传输，卡片消息展示，公众账号查询，详情，消息（地理位置，图片语音）收发功能、群聊功能、群发功能等
    
   - 涉及的技术点
   
       |技术点| 实现方式|
       |-----|--------------------------------------------------------------------|
       |项目结构    | **Jni+AIDL+Service+ContentProvider+broadCast**实现服务进程，采用**多种跨进程通信**方式，给上层短信提供业务能力接口|
       |数据存储    |  使用**SharedPreference、SQLiteDatabase**数据库存储操作（数据库升级）等操作、流文件存储、等方式实现数据持久化|
       |组件化     | 按照业务划分模块，将不同功能组件化，例如鉴权登陆模块、文件传输、群聊消息模块，一对一消息收发模块|
       |Gradle操作 | SO加载JNI配置，gradle操作jar包，**flavor**多分支多渠道打包方式，aar打包|
       |设计模式   | Mms短信代码结构**mvc**模式,Messaging短信代码结构为**Mvp**模式 |
       |定时操作    |JobScheduler +IntentService 实现定时登陆操作 、埋点功能实现，数据上传 |
     
   - 对接的厂商项目
   
        |时间 | 项目 | 内容|
        |-----|-----|----|
        |2019.01-2019.08|**Oppo-外销**Rcs|基于Oppo系统短信源码（MMS），加入RCS-SDK,修改TelephoneProvider，将短信改造为符合GSMA认证的增强信息|
        |2019.08-2019.01|**海信、魅族、小米**-Rcs-maap业务|开发Rcs-maap业务，帮助厂商将Rcs-Maap业务升级，修改原有Rcs业务，完成移动的规范标准升级|
        

#### Juphoon音视频Sdk开发 （2018年4月-2019年1月）

SDK介绍：

主要提供 一对一/多方音视频通话功能,一对一群组Im功能,文件存储功能等。
    
   - 完成的模块
     
     多方通话涂鸦功能修改，三方推送方案实现,多方通话网络状态,查询用户状态,录制分辨率调整等,完成iOS端和windows端代码翻译同步任务。
        
   - 涉及的技术点
      
      |技术点| 实现方式|
      |-----|--------------------------------------------------------------------|
      |项目结构|接口化生成jar包+jni调用媒体库|
      |异步操作方式|Handler+Thread、AsyncTask|
      |开源框架| AgentWeb、Butterknife、Glade、SmartRefresh、EventBus框架、Retrofit2、Realm|
      |界面 | ViewPager+fragment、自定义View、RecyclerView、Activity|
      |自动化测试| python+动态代理+反射+Socket+json |
    
     
   - 使用JC-SDK开发项目
     
     |时间 | 项目 | 内容|
     |-----|-----|----|
     |2018年7月-2018年11月|东莞移动视频营业厅|集成JC-SDK完成视频Android访客端开发。完成大部分UI需求和业务需求|
     |2018年12月-2019年1月| 国盾量子加密通讯 |使用国盾提供加密SDK+JCSDK 完成语音通话，IM消息等功能|
     | 2018年4月-2018年6月 |CoachTalk（展会演示apk）| 使用AgentWeb框架.通过WebView加载H5界面（采用vue框架），在网页界面完成预约功能，调用Native方法完成视频通话和登陆等功能|

## **宁波麦博韦尔移动电话有限公司** （ 2017年3月 ~ 2018年4月 ）

### 工作内容和技能

主要工作内容：

完成Android framework层的客制化需求，修改手机系统测试出现的各种类型BUG。累计解决客户需求和系统Bug 150+。

- 熟悉android_FrameWork层开发，有Debug能力，对代码追踪和问题定位有经验。
- 对短信，SystemUI模块较为熟悉
- 有MTK平台的开发经验

### 超级省电功能 （2017年7月-2017年10月）

参与超级省电需求开发，负责了整个超级省电的基础需求开发和后期优化，完成从入口界面到逻辑处理的整个流程，集中调起并关闭许多耗电模块，并且优化了调起时长，解决某些关闭项之间相互影响，将模块重构并添加宏控，方便后期项目的维护和需求的扩展。对数据库进行了字段的处理，减少了内存的使用。通过延时处理线程问题等。

其他bug：
- wifi和热点相互影响
- 通过其他通道进入耗时界面，屏蔽keydown事件，保存数据状态。


### Safircom项目 

在此项目中负责MMI层的开发工作，完成客户的客制化和需求的移植。修改Modem项，预置APK，修改并通过GTS/CTS认证，修改Framwork层出现的BUG。在此项目中学习了手机系统软件的开发流程。掌握modem编译方式。提高了文档阅读能力。对项目进度和管理沟通，有了深刻的认识。


## 开源项目

 - [服务进程框架方案](https://github.com/cuizehui/ServiceAPK) 使用跨进程通信方式提供服务的框架原型
 - [MVP_App_Frame](https://github.com/cuizehui/MVP_App_Frame)  基于Mvp开发的App-包含常用模块的处理细节和工具类等
 - [BinderPushApk](https://github.com/cuizehui/PushAPK-Binder)  Binder通信实现小米推送apk
 - [SDK自动化测试方案](https://github.com/cuizehui/JCInterFaceTestClient)
 
## 相关技术文章


- [MMS短信源码-重要功能分析](https://cuizehui.github.io/2019/03/22/responsive-android/)

- [进程间通信-Binder实战:https://cuizehui.github.io/2018/07/07/responsive-android/](https://cuizehui.github.io/2018/07/07/responsive-android/) 

- [sdk接口自动化测试:https://cuizehui.github.io/2018/09/25/responsive-android/](https://cuizehui.github.io/2018/09/25/responsive-android/)

- [H5-webViewApp开发总结:https://cuizehui.github.io/2018/05/01/responsive-android/](https://cuizehui.github.io/2018/05/01/responsive-android/)

# 致谢
感谢您花时间阅读我的简历，期待能有机会和您共事。


