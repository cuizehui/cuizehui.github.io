---
layout:     post
title:      "接口自动化测试(三)--动态代理"
subtitle:   "通过自动测试，介绍aop实现和反射注入"
date:       2018-09-28 21:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- AndroidSDK-Test
---

# 接口自动化测试(三)--动态代理回调模块

## 动态代理回调模块

### 简介

[第一篇](https://cuizehui.github.io/2018/09/25/responsive-android/)通过反射测试sdk接口，部分接口会产生回调方法，我们需要将回调结果上传至服务器。

但如果我们手工在各个接口中将返回值依次组json,那么将是一个巨大的工程。
所以我们通过动态代理方式，将回执数据组包，自动发送到socket模块回执给服务器。

### 实验demo

AOPDemo:
[JavaAopDemo](https://github.com/MatrixSeven/JavaAOP)

由于此demo，中没有代理回调。因此我又修改此Demo:

[CallBackProxyDemo](https://github.com/cuizehui/CallBackProxyDemo/tree/master/app/src/main/java/CallBackProxyDemo)

### 动态代理本质 

此方法实现的本质：反射生成一个新的接口对象，实现通过invoke执行原对象所有接口方法，在运行期,反射方法执行的前后插入自定义逻辑。


```java
//绑定委托对象，并返回代理类
public Object bind(Object tar) {
    this.tar = tar;
    //绑定该类实现的所有接口，取得代理类
    return Proxy.newProxyInstance(tar.getClass().getClassLoader(),
            tar.getClass().getInterfaces(),
            this);
}

```

**注意:**
在java中，jdk实现动态代理的主要不足在于,***它只能代理实现了接口的类***，如果一个类没有继承于任何的接口，那么就不能代理该类，原因是我们动态生成的所有代理类都必须继承Proxy这个类，正是因为Java的单继承，所以注定会抛弃原类型的父类。

### 组包代码

最终组包，将对象转为Json也是巨大工程，因此采用了Gson工具将对象转为Json数据。

但在invoke中某些object对象转好只是地址值，因此做了args[i].getClass()特殊处理。

```java
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = null;
        String resultJson = makeResultJson(method.getName(), args);

        method.getParameterTypes();

        result = method.invoke(tar, args);
        sendResult(resultJson);
        return result;
    }

    @Override
    public String makeResultJson(String method, Object[] args) {
        JSONObject resultJson = new JSONObject();
        //此处做处理类型判断处理
        try {
            resultJson.put("type", "callback");
            resultJson.put("method", method);
            Gson gson = new Gson();
            for (int i = 0; i < args.length; i++) {
                if (args[i].getClass() == com.xxxx.JCCallItem.class) {
                    JCCallItem jcCallItem = (JCCallItem) args[i];
                    resultJson.put("arg" + i, gson.toJson(jcCallItem));
                } else {
                    resultJson.put("arg" + i, args[i]);
                }
            }
        } catch (JSONException e) {
            e.printStackTrace();
            return "";
        }
        return resultJson.toString();
    }

    @Override
    public boolean sendResult(String resultJson) {
        if (resultJson == null) {
            return false;
        }
        EventBus.getDefault().post(new JCTestEvent(resultJson));
        return true;
    }
```
---

## 反射注入

在AopDemo的同时发现了此demo:[反射注入框架](https://github.com/ximsfei/RefInject)

其原理为通过定义一个和要反射注入类相同结构的类，通过封装反射方法，获取到注入对象的成员和方法进行动态调用。

### 反射注入简介

在运行期动态的将运行期变量替换，或执行某个方法。
此技术应用于插件化开发。用来替换某个模块或控件等。

## 分析

分析此框架，无法运用到项目中，因为sdk的调用需要通过服务器指令调用。无法使用写死的方式进行。遂放弃。
不过此框架，在运行期替换掉成员变量，可以在处理特殊对象时使用，也就是第一篇文章，中由解析完成特殊对象构造的任务，可以由此技术构造。


## 参考文章

[https://www.jianshu.com/p/c42b3feecb09](https://www.jianshu.com/p/c42b3feecb09)
[https://ximsfei.github.io/2017/07/13/VirtualApp-Java%E5%B1%82Hook%E5%9F%BA%E7%A1%80/](https://ximsfei.github.io/2017/07/13/VirtualApp-Java%E5%B1%82Hook%E5%9F%BA%E7%A1%80/)
[https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/index.html](https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/index.html)