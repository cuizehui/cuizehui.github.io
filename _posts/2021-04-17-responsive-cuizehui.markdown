---
layout:     post
title:      "ANR问题集合"
date:       2021-04-17 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

## ANR 


### APP层如何监控，ANR是如何做监控的？

MainLooper检测

### ANR的产生原理

AMS发一个任务消息,再次发送一个延时ANR消息，正常情况下执行任务消息后ANR的消息会被移除。没有移除则出现ANR

mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);

- ANR是何时开始计时的

  生命周期回调，所以可以在这个逻辑入口开始计时。
  
  provider的超时是在provider进程首次启动的时候才会检测，当provider进程已启动的场景，再次请求provider并不会触发provider超时

### ANR 出现的原因，如何定位  

比如前台服务在20s内未执行完成

### 如何避免

- 16ms屏幕刷新率，可即使监听Input事件

### 常见ANR问题

1. 读写小文件 IO资源调度 IO Wait
2. Binder通信
3. 主线程死锁
4. CPU饥饿状态 LOAD
5. Input事件，响应和焦点问题。

### 目录日志

/data/ANR/trace

将am_anr信息输出到EventLog，也就是说ANR触发的时间点最接近的就是EventLog中输出的am_anr信息

收集以下重要进程的各个线程调用栈trace信息，保存在data/anr/traces.txt文件

当前发生ANR的进程，system_server进程以及所有persistent进程

audioserver, cameraserver, mediaserver, surfaceflinger等重要的native进程
CPU使用率排名前5的进程
将发生ANR的reason以及CPU使用情况信息输出到main log
将traces文件和CPU使用情况信息保存到dropbox，即data/system/dropbox目录
对用户可感知的进程则弹出ANR对话框告知用户，对用户不可感知的进程发生ANR则直接杀掉