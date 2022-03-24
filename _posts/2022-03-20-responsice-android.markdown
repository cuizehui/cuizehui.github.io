---
layout:     post
title:      "将依赖的远程库改为本地依赖"
date:       2022-03-20 16:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---


## 简介

有时项目会依赖远程库，在离线模式如果清除缓存缓存,并且不能访问远程仓库,则必然编译不过。此时先需要将远程依赖改为本地依赖以确保项目编译通过.这有什么用,懂得都懂。

### 远程私有仓库种类

 Nexus 称为“Maven仓库管理器
 
 Maven 包（Package）

jcenter 远程仓库
至于 Maven 是什么，请参考 Apache Maven。

对于 Android 开发者而言，只需要知道 Maven 是一种构建工具，Maven 包是由所谓 POM（Project Object Model）所定义的文件包格式即可。

Gradle 可以使用 Maven 包，而且大部分的 Android 能够使用的远程依赖包都是 Maven 包。

### 非公开仓库的使用

```
repositories {
        maven { url "http://maven地址" }

        google()
        jcenter()
       
    }
    
dependencies {
    api 'maven 仓库中的库'
    api 'androidx.startup:startup-runtime:1.1.0'
    api 'org.greenrobot:eventbus:3.2.0'
}

```

### 三方库下载本地路径

gradle同步后,校验成功,AS会去远程maven拉取到本地

在AndroidStudio中的"External Libraries"下有引用的library的列表 ,右键Library Properties ..找到临时缓存路径
此地址非原始jar或者aar 地址
真实地址在 **.gradle/caches/modules-2/files-2.1**


### 将aar或jar依赖导入

step1:
    把aar和jar都丢在libs下面

```
    
    android {
        
        repositories {
            flatDir {
                dirs '../libs'
            }
        }
       
    }
```

step2:

```
     api (name: 'aarname', ext: 'aar')
```
    
api 表示依赖这个moudle自动可以引入 此aar 但是上面的路径不能省略


从时即可将远程maven依赖去掉了。当然这种方法,没法同步远程最新库。

## 参考

https://blog.csdn.net/T_yoo_csdn/article/details/80016601
https://stackoverflow.com/questions/68344424/unrecognized-attribute-name-module-class-com-sun-tools-javac-util-sharednametab