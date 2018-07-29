---
layout:     post
title:      "Activity启动模式和近期任务卡"
date:       2018-07-28 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

# Activity启动模式和近期任务卡

## 简介

此文讲了，四种启动模式中存在的坑。 并讲述了和启动模式栈和近期任务卡之间的关系。

## 四种启动模式和坑

可以通过此命令查看栈 和activity的情况

```
adb shell dumpsys activity
```

### standard

要注意的是此模式在5.0之后，如果开启的activity不属于同一个应用内，则近期任务会显示两张任务卡


### signletop
 
 只存在同一个栈，如果在栈顶则不会创建新的实例。
 
### singleInstance 

使用此启动模式的Activity会独占一个Task 
startActivity方法启动的activity也会出现在一个新的栈,被start也和调用者不在同一个栈.

**坑1：** 近期任务只会显示一个最近打开的task，返回键无法回到最初的task

如何显示多个近期任务卡呢？

使用android:taskAffinity=“不同于当前包名的包名" 则可以显示多个task

**坑2：** sigleInStance 的activity调用finish后,栈并未销毁.从近期任务点进去，会发生其他错误。

使用finishAndMoveTask() 将栈和activiy同时销毁，此方法5.0以上可用。


 
### singleTask 

只存在一个栈，栈内只有某种Activity的一个实例

**坑 ：** 当存在此实例，则不会调用onCreate方法，并在栈中将此activity上的其他activity都移除，以达到将此singleTask的activity放到栈顶的目的。

[https://github.com/fred-ye/summary/issues/46](sigleTask坑)

## 参考文章

https://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/

https://www.imooc.com/article/23757

https://github.com/fred-ye/summary/issues/46

