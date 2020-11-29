---
layout:     post
title:      "Rcs技术方案总结"
subtitle:   "Rcs电信培训演讲稿和PPT"
date:       2020-01-21 15:50:00
author:     "Nela  "
header-img: "img/post-bg-rwd.jpg"
tags:
    - Nela
---

# 技术方案总结
Apk分为两部分 一个应用进程，一个服务进程

## 服务进程

### 工程结构

1.common提供jar包通过AIDL调用service方法和SO库方法
2.module封装service、provider、Receiver进行数据库操作数据库和事件上报
3.jniLibrary 封装c++接口文件,生成So库和配套Jar包提供module使用

### aidl接口方法向下调用方式：
1.ServiceManager.init注册所有service 
2.ServiceManager.addCallBack 注册service回调事件 包括Native方法回调
3.通过CallWrapper接口调用service方法

### 广播和回调方式：

所有广播和回调注册在Application中：
1.收到广播后 统一调用Application.dealFirstNotify()上报一次广播，开启IntentService处理广播事件，处理完毕后如果仍需要上层处理则给Apk层的广播发送消息
一次广播处理完毕，看业务,可通过回调或者广播上报给APk


### 增加签名文件和签名文件配置

   
## 数据库操作

### provider 

1. 定义操作表名
2. 定义Uri 路径
3. 获取DataBaseHelper
//数据库升级逻辑
4. 根据UriMatch 分配操作的数据表
//待完善 增删改查 


---------




