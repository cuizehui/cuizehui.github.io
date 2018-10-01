---
layout:     post
title:      "Android—自动编译"
date:       2018-09-12 23:18:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---、	

# 自动编译

## 简介
 
 本文介绍Jenkins+git平台下，Android脚本处理库文件，通过gradle进行的自动编译
 
## 重要性

我认为一个构建应该起码包含运行你的单元测试。
因此，单元测试现在成为了你和你所引入的bug之间的第一道防线。我甚至建议只有一个人的团队也要搭建自动构建系统。 

## jenkins流程

流程如下：钩子脚本触发构建(jenkinsfile)-拉源码(git插件配置)-> 执行本地编译脚本(build.sh)->编译构建(gradle/sdk/jdk环境)->生成构建成果

### 基本模式

1. 安装插件
2. 全局配置 jdk gradle git 路径位置，包括配置没有的环境变量等
3. 访问指定地址（uri）地址既可以进行触发构建
3. 触发构建脚本
4. 编写本地shell脚本执行本地构建工具
5. git外网环境，webhook.

### Pipeline 流水线

jenkinsfile

```
        extensions: [
        [$class: 'CleanBeforeCheckout']
       ],
```
                 
pipeline出现，它就是jenkins部署的代码方式，它使用groovy脚本编写，有了它，就不用使用jenkins向导了，有了它，你的jenkins变更就可以追踪了（因为它的文件可以放在git,svn上）。

## gradle 编译

### 认识gradlew
基础知识： gradlew和gradle不是同一个东西。
参考此篇文章
[https://juejin.im/post/5ac9d48d6fb9a028e014bf15](https://juejin.im/post/5ac9d48d6fb9a028e014bf15)

**问题 1**

```
Could not generate a proxy class for BuildArtifactReportTask

```
发现是gradle 版本不对

### 基础编译命令

```
./gradlew clean
./gradlew assembleRelease //编译release版本

```
### 通过gradle签名

1. 生成签名文件
2. gradle 签名 

 ```
 
    signingConfigs {
        signConfig {
            storeFile file("../keystore.jks")
            storePassword STORE_PASSWORD
            keyAlias KEY_ALIAS
            keyPassword KEY_PASSWORD
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.signConfig
        }
    }

 ```

3. 在./gradle.properties 中添加常量值

rm -rf ./gradle.properties

## build.sh

### 移动库文件

有些项目需要的底层库需要shell脚本移动
 ```
mkdir -p Android/app/libs/armeabi-v7a
mkdir -p Android/app/libs/x86

cp -f shared/service/all/android/lib/android/*.jar VideoCallCenter/app/libs/
cp -f shared/service/all/android/lib/android/armeabi-v7a/*.* Android/app/libs/armeabi-v7a
cp -f shared/service/all/android/lib/android/x86/*.* Android/app/libs/x86

cd Android
```

### 打包工程源码

去除build文件和编译脚本等无用文件
去除appKey

```
if [[ "$1" == version:* ]];then
	
	sed -i 's/\"appkey"/输入AppKey/g' ./app/src/main/java/com/juphoon/vccclient/VccConstants.java
	
	tar -zcvf ./out/CallCenter-Android.tar.gz  --exclude=./app/build --exclude=./app/gradle.properties  ./app  build.gradle gradle.properties settings.gradle ./gradle gradlew gradlew.bat
fi
```

