---
layout:     post
title:      "Android逆向工程初探"
date:       2018-05-19 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---
# Android逆向工程初探
## 准备工作
参考如下博客：
[https://blog.csdn.net/yanzi1225627/article/details/48215549]()
## 反编译
使用apktool 

反编译命令

```
apktool d xxx.apk 
```

## 修改源码

### 查看源码
- dex2jar将apk转成相应jar包
工具所在目录下执行

```
	./d2j-dex2jar.sh  usr/~~~~/apk(apk路径)
```

- 将生成的jar包丢进打开的JD-GUI工具中

### 源码情况
- 裸奔
- 混淆
- 加固
- 有签名防护

### 修改源码smali

#### 常见语法
待更新

## 二次打包
命令

```
apktool b xxx
```

xxx为反编译后的文件
dist文件夹下会生成未签名的apk 


## 签名

1. 生成签名文件
	
	```
	keytool -genkey -keystore nelakeystore -keyalg RSA -validity 10000 -alias nelakey
	
	```

2. 签名
	
	```
	jarsigner  -verbose -keystore nelakeystore -signedjar target-sign.apk target.apk  nelakey
	
	```

其中有签名时效等等问题。

## 参考
[https://blog.csdn.net/zgzczzw/article/details/52669215](签名防护方案)

[https://www.jianshu.com/p/53078d03c9bf](签名相关)


