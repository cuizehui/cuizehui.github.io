
# 联系方式

- 手机：18668579239    
- Email： 454776824@qq.com   
- WeChat：czh34351

---

# 个人信息

 - 崔泽辉/男/1994 
 - **本科/华东交通大学** 物联网工程系 
 - 工作年限：**2年**
 - 期望职位：Android应用开发工程师
 - 技术博客：[https://cuizehui.github.io/](https://cuizehui.github.io/) 
 - GitHub：[https://github.com/cuizehui](https://github.com/cuizehui)
 - CSDN博客：[http://blog.csdn.net/cuizehui123](https://blog.csdn.net/cuizehui123)
 
 ---

# 个人技能简介

- 有音视频SDK开发经验，RCS开发经验
- 有基于系统短信（MMS）源码开发的经验
- 有基于Vue+AgentWeb框架的混合App开发经验
- 熟悉Git分支操作,熟练掌握AndroidStudio使用,熟悉Gerrit+Jenkins平台,熟悉Gradle
- 熟练掌握Mac/Ubuntu/Windows系统下的开发操作
- 有良好的代码规范,命名规范
- Java基础基础良好,掌握正则,Socket,反射等技巧。掌握常用设计模式：单例、享元、工厂，原型，动态代理等,能设计并使用mvc/mvp框架进行开发
- Android基础良好,能够完成大多数界面开发,并快速学习UI控件使用,对Binder进程间通信有过开发经验
- 能够编写简单Python脚本，对JavaSDK接口自动化测试有搭建经验

# 工作经历

## **宁波菊风系统软件有限公司**（ 2018年4月 ~ 至今 ）

### Oppo-Rcs项目 （2019年1月-至今）

**项目需求**：基于Oppo系统短信源码，加入RCS-SDK,修改MMS,TelephoneProvider，Contact等,将短信改造为符合GSMA认证的增强信息。

    独立完成的功能模块：
        
        - 群聊功能 群头像/群群管理
        - 群发功能（多个一对一模式）/（群发转短数据库操作）
        - 界面相关，编辑模式下搜索，短信RCS模式切换等 ActionMode+Toolbar

#### JuphoonCloudSdk （2018年4月-2019年1月）

SDK介绍：主要提供 一对一/多方音视频通话功能,一对一群组Im功能,文件存储功能等。
    
    参与并完成以下功能和任务
    
    - 封装媒体层和信令层So库接口,并迭代新功能（查询用户状态,多方通话网络状态,录制分辨率调整等）
    - 集成新Push模块,Push模版组建，Portal迭代,三方平台(小米,华为,FCM,Oppo)Push协议适配,Push异常错误码处理,并发请求排队机制
    - 搭建自动化测试（python+动态代理+反射+Socket+json）
    - 客户需求支持

#### 加密通讯QTalk-Apk(2018年9月-2018年12月)

此项目是一个在Linux系统上运行的Apk,主要是调用JuphoonSDk完成语音通话,IM消息等功能

    参与并独立完成以下功能：
    
    - 使用mvp设计模式进行开发
    - 封装Juphoon-SDk,完成语音通话，IM消息，语音消息，文件传输等功能。
    - 使用底层Binder机制,配合C层完成Push功能
    - 使用EventBus,Retrofit，Realm等开源项目辅助完成开发
    - 登陆采用oauth2.0鉴权模式完成登陆功能

#### CoachTalk-APK(2018年4月-2018年6月) 
 
是一款通过在APK内H5(采用vue框架)页面完成在线预约功能，并通过页面调用Native的登陆，通话等功能的一款混合APP

    参与并完成以下功能
    
    - 封装Juphoon-SDK到agentWeb结构中，给Html界面提供登陆,通话等能力接口
    - 使用AgentWeb,Butterknife,Glade,SmartRefresh等开源项目和技术辅助完成发开发
    - 调用通过ContentProvider,调用Calender接口,完成日历提醒功能。


## **宁波麦博韦尔移动电话有限公司** （ 2017年3月 ~ 2018年4月 ）

### 工作内容和技能

主要工作内容：

完成Android系统的客制化需求，修改手机系统测试出现的各种类型BUG。累计解决客户需求和系统Bug 150+。

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


