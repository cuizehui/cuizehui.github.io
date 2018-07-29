---
layout:     post
title:      "Android进程间通信-Binder实战"
date:       2018-07-07 21:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

# Android进程间通信-Binder实战

## 简介

本文记录了一次由JAVA层和C层通过Binder完整且稳定通信的方法。

首先介绍了Binder通信的基本原理，然后通过需求切入，提供了两种通信方案（即java层注册service和c层注册service）然后根据实际情况，实现方案。并记录了实现过程中出现的问题和解决思路。

## 基本概念

Binder通讯数据复制一次的最本质原因：

得益于 Linux 的动态内核可加载模块（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。
在 Android 系统中，这个运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 Binder 驱动（Binder Dirver）。


内存映射：
Binder IPC 机制中涉及到的内存映射通过 mmap() 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。

sercice注册服务-通过binder对象连接-servicemager和用户空间
并做内存映射。以达到让用户空间访问servcie服务的方法。
 
**小结:**
理清思路，即service端产生Binder服务。通过serviceManager注册至系统内核. client端通过serviceManager获取Binder服务（实际是代理对象）完成通信。

## 需求

实现如下需求：

linux进程服务想要获得推送功能。但c层无法实现。实现思路是通过安装apk并通过三方平台注册push功能。然后在系统开启时使apk变为常驻服务。当apk收到推送消息时，能通过apk所在进程将将消息传递给linux服务进程。

那么实现进程间通信的方式就选择了上述所说的Binder机制。

实现前首先考虑以下几个问题：

1. linux进程死亡Binder失效，如何回调通知apk？
2. Binder机制是不是只能单向通信，即linux只能调apk方法？如果是单向通信该如何解决。  

## 实现思路

由于不知道如上两个问题如何解决。不知道是否是单向通信，那么我们确保通信的稳定。则要按照如下步骤进行实现。

1. c层实现service,java层实现client
2. java实现service,c层实现client

## 具体实现

### c层实现service

**第一步.** 生成Binder服务：

```
android::status_t CJuBinder::onTransact( uint32_t code, const android::Parcel& data, android::Parcel* reply, uint32_t flags)
{
    Mtc_AnyLogInfoStr((ZCHAR*)"CJuBinder", (ZCHAR*)"CJuBinder::onTransact code:%d", code);
    switch (code)
    {
        case 1:
            {   
                peerCb = data.readStrongBinder();
                NotifyPeer(1);
            }
            break;
        case 2:
            /* 收到push，唤醒终端 */
            MyProcNotify(MtcActionWakeupThread, 0, NULL, 0);
            break;
        default:
            return android::BBinder::onTransact(code, data, reply, flags);
    }

    return 0;
}

```
**代码讲解：**
onTansact为服务端需要实现的方法，其中code是service和client约定好方法标识，data为客户端传递参数。reply为返回值。


**第二步.** 注册binder服务至系统内核

```
int CJuBinder::RegisterService()
{
    android::sp<android::IServiceManager> sm = android::defaultServiceManager();
    android::status_t ret;
    ret = sm->addService(mydescriptor, this);
    Mtc_AnyLogInfoStr((ZCHAR*)"CJuBinder", (ZCHAR*)"OnBinderLoop addService ret:%d", ret);
    //call binder thread pool to start
    android::ProcessState::self()->startThreadPool();
    android::IPCThreadState::self()->joinThreadPool(true);
}

```

**代码讲解：** addservice方法中的mydescriptor 是service名字，client端也是通过这个名字查找binder的

***特别注意***上述代码均是由c层同事编写。可能粘贴有不准确或遗漏的部分。可参考此网站：
[https://blog.csdn.net/ganyue803/article/details/41315733](网上事例代码)

### Java实现client

**第一步.**获取Ibinder对象

***问题：***
首先通过ide发现，android并没有serviceManager这个类。那么我们通过反射获取serviceManager对象并调用getService方法获取IBinder.


```
 private static IBinder getRemoteBinder() {
        try {
            Class serviceManager = Class.forName("android.os.ServiceManager");
            Method method = serviceManager.getMethod("getService", String.class);
            mRemoteBinder = (IBinder) method.invoke(serviceManager.newInstance(), "yourdescriptor");
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }
```

**第二步.**调用方法


```
public boolean sendTokenAndPackageName(Context context, String token) {
        Log.d(TAG, "enforceInterface");
        mContext = context;
        mToken = token;
        JSONObject jsonObject = new JSONObject();
        try {
            jsonObject.put("token", token);
            jsonObject.put("packName", context.getPackageName());
            jsonObject.put("through", 1);
        } catch (JSONException e) {
            e.printStackTrace();
        }
        String pushInfo = jsonObject.toString();
        Log.d(TAG, "token " + jsonObject.toString());
        Parcel _data = Parcel.obtain();
        Parcel _reply = Parcel.obtain();
        _data.writeString(pushInfo);
        _data.writeInterfaceToken("nela.AndroidPush");
        boolean result = false;
        try {
            result = mRemoteBinder.transact(1, _data, _reply, 0);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        _reply.readException();
        _reply.readInt();
        return result;
    }

```

**代码讲解：** 
调用c层提供给我们的方法，code为int型，我们定义方法为1。
这里我们参数传递了string类型字符串，由于要传递的string很多，那么我们通过定义json数据格式传递。

***注意：***_data.writeInterfaceToken() 这个方法用于标识的打包对象，和_data.enforceInterface()方法为一对，也可以不填加此标识。

***遇到的问题：***
上述代码传递字符串发生了c层获取字符串乱码的问题，c层通过getCstring.获取不到或者不对。可以用data.size 判断java层是否将数据传递至c层。每次尝试时，记得将apk卸载重试。原因不详。

***小结：***
调试结果,c层可以拿到参数并调用相应方法。至此c注册服务-android获取并调用的流程就已经简单打通。

但是为了解决最开始的两个问题，那么java层也要注册服务，c层获取并调用。

### JAVA层实现service 

**第一步.**实现Binder，继承binder对象

```
public class LocalBinder extends Binder implements IPushInterface {

    public static String TAG = "LocalBinder";

    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        switch (code) {
            case 1:
                data.enforceInterface("Juphoon.AndroidPush");
                sendTokenAndPackageName();
                Log.d(TAG, "onTransact getTokenAndKey");
                break;
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public void sendTokenAndPackageName() {
        Log.d(TAG, "LocalBinder" + "sendTokenAndPackageName");
        BinderManager binderManager=BinderManager.getInstance();
        binderManager.sendTokenAndPackageName(context, token);
    }

    @Override
    public IBinder asBinder() {
        return this;
    }
}
```

实现IInterface接口

```

public interface IPushInterface extends IInterface{

    public void sendTokenAndPackageName();

}

```

**代码讲解：** binder对象主要是重写***onTransact*** 方法。IInterface 接口重写 ***asBinder()***。这两个方法是最重要的。也是android中AIDL实现的基本原理。在AIDL自动构建好的文件中上述这个LocalBinder被叫做stub.

***延伸***：aidl是在这个基础上做了有一层封装，aidl中还有一个queryLocalInterface和attachInterface。

attachInterface也是接口标示，而queryLocalInterface则和我们后面说到的传递Ibinder对象有关，会判断是否是本地binder.


**第二步.** 注册Binder

binder创建好后则需要通过ServiceManager注册至系统内核，同理也是通过反射调用addService：

```
   try {
            Class serviceManager = Class.forName("android.os.ServiceManager");
            Method method = serviceManager.getMethod("addService", String.class,IBinder.class);
            LocalBinder localBinder=new LocalBinder();
            method.invoke(serviceManager.newInstance(),"Myservice",localBinder.asBinder());

        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
```

**调用结果：** 系统报出异常 **InvocationTargetException** 查阅资料是内部异常，最终堆栈信息打印的是java.lang.SecurityException。 

***问题：*** java.lang.SecurityException异常处理

查看serviceManager类发现：
并不是所有的注册请求都会得到响应，当发出请求的进程具有相应的权限时，才会做出相应操作。权限可以分为以下2种：
  
  - 特权。具有root和system权限的进程可以注册任何服务
  - 普通权利。对于一般进程，需要查看该进程是否有权限注册它所请求的Service。每个Service对应有一个uid表
  
参考： [https://blog.csdn.net/wlwl0071986/article/details/50977573]()

***解决方案：***

- 将apkpush至系统system/priv-app 使其变成系统apk，再次尝试
- 删除旧的系统apk 需要权限，需执行如下代码获取读写权限 mount -o remount rw  /system

具体过程
参考如下文章：
[https://blog.csdn.net/starhosea/article/details/78697007]()
最后仍然报出如上异常。

- 解决方法2
第一步：在AndroidManifest.xml中，添加 android:sharedUserId="android.uid.system"
第二步：使用系统签名后再安装即可。***此方法未实验***

**总结：** 　Android 层可能无权限,将binder注册通过serviceManager注册至系统内核，那么如何确保通信的稳定和双向通信的问题呢?

### Binder死亡代理

通过IBinder提供的api：linkToDeath/unlinkToDeath

```
mRemoteBinder.linkToDeath(mdeathRecipient, 0);
 
IBinder.DeathRecipient mdeathRecipient = new IBinder.DeathRecipient() {

        @Override
        public void binderDied() {
            mRemoteBinder.unlinkToDeath(mdeathRecipient, 0);
            Log.d(TAG, "mdeathRecipient");
            mRemoteBinder = null;
            mRemoteBinder = getRemoteBinder();
        }
    };
```

当服务进程死掉时会回调binderDied；

测试方法：

```
adb shell
ps | gerp "进程名"
kill 进程id
```

### 双向通信

通过Binder类发现 client 通过transact方法传递的参数data,为Parcel类
将数据打包发送和序列化相关。

其中writeStrongBinder() 允许传递一个Binder，

service通过readStringBinder()即可获取。并调用传递binder定义的接口方法完成回调。

```
_data.writeStrongBinder(localBinder.asBinder());
```

***问题：***传入binder获取不到.

***解决：***要调用asBinder 将自定义的binder传递进去，强转或者new 对象都无效！！

那么我们这里就把上述的本来要在java层要注册的Binder对象直接专递进去，即完成了双向通信。

### 延伸问题

AIDL 为什么能注册 ？

因为它底层通过service-》activityServiceManager注册至系统内核
activityServiceManager是系统内核控制的。

### 后续完善

判断网络&重连机制

### 代码地址
[https://github.com/cuizehui/PushAPK-Binder]()

## 参考文章

https://blog.csdn.net/carson_ho/article/details/73560642

https://www.cnblogs.com/hpboy/archive/2012/07/12/2587797.html

https://www.cnblogs.com/zhangxinyan/p/3487866.html

https://blog.csdn.net/ganyue803/article/details/41315733