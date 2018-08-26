---
layout:     post
title:      "Android—push方案总结"
date:       2018-08-28 14:18:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

# Android—push方案总结

## 本文简介

介绍push的几种方案，针对长链接方案，了解一种思路push实现的思路，并介绍sdk层如何将push和业务模块相互结合。

## push原理

1. 长链接方案 

长链接是为了 数据传输和推送。保证客户端和服务端一直处于通信状态。

一种是app层的长链接，由apk自己维护，需要做进程保活断线重连等等。

一种是系统级长链接，该进程由系统维护，例如小米手机，则维护了小米推送的长链接。

系统级别的长链接，收到push消息后有两种选择，一种是交给notifiaction处理，一种是下发至app处理，也就是透传和非透传。所以透传消息的到达率是低于非透传消息的。因为透传消息走的也是app级别的通道。

小米推送华为推送等如果在其对应的手机上走的都是系统及长链接。而像个推，极光推送等走的都是app级别的长链接。在三方推送中，其提供的通知和透传消息走的都是app级通道。


2. SMS信令推送

服务器有新消息时，发送1条类似短信的信令给客户端，客户端通过拦截信令，解析消息内容 / 向服务器获取信息  此种方案基本可以理解为付费通道。

3. 轮询

定时拉取服务器消息。

心跳和轮询的区别，心跳是建立在已经有的链接上，轮训需要经历一次tcp+断开，3次握手4次挥手

## 长链接的实现

### BIO和NIO

传统的实现一个长链接的方式是BIO.也就是阻塞式，更合理的实现方式为NIO.

BIO模型分析:
传统阻塞线程io
需要创建线程 
线程的创建和销毁成本很高，在Linux这样的操作系统中，线程本质上就是一个进程。创建和销毁都是重量级的系统函数。
线程本身占用较大内存，像Java的线程栈，一般至少分配512K～1M的空间，如果系统中的线程数过千，恐怕整个JVM的内存都会被吃掉一半。
线程的切换成本是很高的。操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。如果线程数过高，可能执行线程切换的时间甚至会大于线程执行的时间，这时候带来的表现往往是系统load偏高、CPU sy使用率特别高（超过20%以上)，导致系统几乎陷入不可用的状态。

NIO:

NIO同步非阻塞
BIO里用户最关心“我要读”，NIO里用户最关心"我可以读了"，在AIO模型里用户更需要关注的是“读完了”。

NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步阻塞的（消耗CPU但性能非常高）。

```
  interface ChannelHandler{
      void channelReadable(Channel channel);
      void channelWritable(Channel channel);
   }
   class Channel{
     Socket socket;
     Event event;//读，写或者连接
   }

   //IO线程主循环:
   class IoThread extends Thread{
   public void run(){
   Channel channel;
   while(channel=Selector.select()){//选择就绪的事件和对应的连接
      if(channel.event==accept){
         registerNewChannelHandler(channel);//如果是新连接，则注册一个新的读写处理器
      }
      if(channel.event==write){
         getChannelHandler(channel).channelWritable(channel);//如果可以写，则执行写事件
      }
      if(channel.event==read){
          getChannelHandler(channel).channelReadable(channel);//如果可以读，则执行读事件
      }
    }
   }
   Map<Channel，ChannelHandler> handlerMap;//所有channel的对应事件处理器
  }

```

NIO等方案，实现起来比较复杂。可以使用框架MINA等第三方框架实现长连接。该框架实现了心跳包，断线重连的基础逻辑。

## 进程保活

完成长链接后重要的问题就是进程保活。由于不是系统级的进程，因此非常容易被杀死。

通常的手段为：监听广播，应用间广播互相拉起

## sdk集成push

我司集成push，主要采用的方式为集成三方厂商的长链接和普通app接入三方推送并无差异。
流程如下：
							->业务服务器
推送模版+token->推送服务器													->三方长链接服务器 上行注册

客户端->业务服务器->推送服务器->三方长链接服务器 判断是否注册 发送下行消息/上行消息

由业务服务器决定，push是否发起，当push消息需要发送时，先发送给推送服务器，推送服务器根据之前约定好的payload ,替换占位符，在发送给三方推送服务器。最终下发至客户端，走系统长连接。


## 参考文章

https://tech.meituan.com/nio.html

https://www.jianshu.com/p/b61a49e0279f

https://blog.csdn.net/woorh/article/details/9854779

https://blog.csdn.net/carson_ho/article/details/79522975

https://blog.csdn.net/MCshidi/article/details/51590297
