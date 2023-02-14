---
layout:     post
title:      "内存优化问题"
date:       2022-02-05 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

## 简介

本文介绍内存泄漏的常见问题

## 常见问题

- 内存泄漏如何检测？
   
   AndroidProfile  Mat

- 内存泄漏是如何产生的，本质是什么？
    长生命周期对象持有短生命周期对象，且没有释放，导致短生命周期对象无法被GC回收

- 静态匿名内部类和匿名内部类有什么区别？
    静态匿名内部类不会持有外部引用

- 弱引用,强引用,软引用？

    - 强引用当对象无指向时，会被GC
    
    - 软引用当内存不足的时候会被回收 会被回收当作缓存
    
    - 弱引用GC回收时，弱引用来引用短生命周期对象ThreadLocal()
        - 线程类的成员变量,ThreadLocal线程本地对象
        - ThreadLocal作用 ThreadLocal中包含map,当map的key是ThreadLocal对象，key使用的是弱引用
        - 避免当ThreadLocal=null时 map仍指向ThreadLocal导致泄漏。但是value仍存在内存泄漏情况。
        - ThreadLocal可以存大对象吗？
    - 虚引用 当一个虚引用对象被回收时,信息会被通知到虚引用队列中,虚引用get找不到对象。
    
## 应用占用内存排查方式

    ```
    adb shell 
    dumpsys meminfo packageName
    ```
## 测试

```bash
adb shell 
dumpsys meminfo packageName
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f5d57d68-66b9-48f9-bacc-013cae3c87f1/Untitled.png)

获取内存分配的更详细内容

获取**smaps文件**

```bash
adb pull /proc/pid_of_app/smaps 
```

获取内存分配的详情

[https://www.jianshu.com/p/07c476705245](https://www.jianshu.com/p/07c476705245)

## 测试数据分析

dumpsys meminfo  获取信息如上图

统计了PSS ,RSS ,Private clear  ,Private dirty ,Swap dirty

这几个数据的关系如下

[https://developer.android.com/topic/performance/memory-management](https://developer.android.com/topic/performance/memory-management)

因为低内存时`kswapd` （守护进程）会将 部分内存移动至zRAM(swap区) 和清除clear 页。以减少内存占用。

共享内存是按照比例分配进行统计的

## 分析思路

```bash
Java Heap:     7480                          36832
         Native Heap:    24300                          28100
                Code:    19588                         146868
               Stack:     2832                           2848
            Graphics:    10448                          10448
       Private Other:     4004
              System:     9673
             Unknown:                                   16660
```

```bash
** MEMINFO in pid 11178 [com.transsion.smartmessage] **
                   Pss  Private  Private     Swap      Rss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty    Total     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------   ------
  Native Heap    23434    23364        0        0    27176    34240    26306     3111
  Dalvik Heap     7548     7352        0        0    16240    14684     7342     7342
 Dalvik Other     5087     4660        0        0     8348
        Stack     2876     2876        0        0     2888
       Ashmem       29        0        0        0      576
    Other dev       25        0       24        0      492
     .so mmap     6588      416      132        0    70268
    .jar mmap     4881        0      312        0    48416
    .apk mmap    42766      244    41152        0    47916
    .ttf mmap      570        0      272        0     1056
    .dex mmap       19        8        0        0      660
    .oat mmap       82        0        0        0     2604
    .art mmap     7295     6988        0        0    21468
   Other mmap      993       48        8        0     5864
    GL mtrack    10376    10376        0        0    10376
      Unknown      800      792        0        0     1600
        TOTAL   113369    57124    41900        0   265948    48924    33648    10453
```

1. Native内存包括共享库
2. Dalvik Heap 虚拟机内存 共享内存分配
3. stack 线程内存
4. so 加载so库
5. jar?
6. apk mmap?
7. dex ?
8. art ?
9. oat ?
10. Gl mtrack ?
11.

[https://ivonhoe.github.io/2019/07/08/how-to-analyze-android-heap-1/](https://ivonhoe.github.io/2019/07/08/how-to-analyze-android-heap-1/)

[https://zhuanlan.zhihu.com/p/95626955](https://zhuanlan.zhihu.com/p/95626955)

分析发现，业务申请的内存主要分布在两部分，即:

- 程序文件:
    - 占比：应用的.dex文件（占35%）、加载的so文件（占7%）
    - 解决方案：
        - **删减代码，减少dex文件内存占用：**K歌内有许多旧代码实际已经废弃，可以将其删除。这里的优化是一个持续的过程，其中歌房在一次代码整理之后就删除了8W多行代码。
        - **按需加载,减少so文件内存占用：**通过smap文件发现，部分加载到内存的so实际是无需运行的，例如直播观众端不需要使用美颜相关的功能，这里可判断仅主播端才加载相关so，大大减少so文件带来的内存开销。
- NativeHeap:
    - 占比：业务集成的so库内申请和图片（8.0及以上版本）共占39%
    - 解决方案：检测、监控申请内存的业务，修复不合理申请。
    - 面临困难：不像Java内存有系统支持的内存快照文件，分析起来非常方便。**Native内存分析就显得非常困难，后续内容将围绕So库与图片内存问题展开叙述。**

  ## Native内存分析

  ### **2)、Native内存申请流程**

  如下图,当应用APP申请内存时，申请的是虚拟内存。当应用访问这块内存并进行写操作时，如果物理内存还未分配则会发生缺页中断并触发分配物理内存。在实际分配物理内存时，是以“页”为单位，每页通常是4KB的内存空间。完成分配后，在MMU模块中的PageTable记录了每一页的虚拟地址和物理地址映射关系。
    
## 内存泄漏检测工具


### LeakCanary工作原理  

弱引用 引用队列，手动GC，如果还是在队列是发生了内存泄漏
 
 
    