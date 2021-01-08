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

## 简介

本文介绍，如何使用APKtool反编译apk，使用dex2jar和jd-gui查看源码，并使用签名工具，对修改后的apk进行二次签名。

如何配置smali调试环境，动态调试和修改smali源码。并简单介绍了smali的语法和代码调试跟踪思路。

最后介绍了apk内部数据库如何获取。

## 准备工作

参考如下博客：
[https://blog.csdn.net/yanzi1225627/article/details/48215549]()

## 反编译

使用apktool 

反编译命令

```
java -jar apktool.jar d xxx.apk 
```


## 查看源码

### 将apk->jar包

- dex2jar将apk转成相应jar包
工具所在目录下执行

```
	./d2j-dex2jar.sh  usr/~~~~/apk(apk路径)
```

- 将生成的jar包丢进打开的JD-GUI工具中
https://blog.csdn.net/yanzi1225627/article/details/48215549

### 源码情况

- 裸奔
- 混淆
- 加固
- 有签名防护
- 有反调试
- smali文件故意制造语法错误导致部分失败

## 修改源码smali

### smali语法

[https://blog.csdn.net/lostinai/article/details/48975661](基础语法)

### 调试smali

#### smali调试环境

1. smalidea插件地址:[https://bitbucket.org/JesusFreke/smali/downloads/]()
2. 准备反编译后的smali工程,manifest中开启appdebug

	```
	<application android:debuggable="true" tools:ignore="HardcodedDebugMode"  
....  
.... />  
	```
	开启debug 后可在logcat打印查看。

3. 添加romete调试选项，并开启手机调试功能。
	
	```
	提示如下： 下午2:53	* daemon not running; starting now at tcp:5037
	```
	
4. 打开Android monitor查看apk对应端口号（右侧）
5. 修改调试remete调试端口 开启debug
  
   ```
   Connected to the target VM, address: 'localhost:8601', transport: 'socket'
   ```
   
#### 静态插桩法调试

通过插入log 查看变量值或者跟踪代码逻辑

```
 插入两个字符串log :
  
	 ````
	    const-string/jumbo v0, "czh"
	
	    const-string/jumbo v1, "sss"
	
	    invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
	
	 ````

```
变量强制转换代码：

```
invoke-static {v1}, Ljava/lang/Integer;->toString(I)Ljava/lang/String;  

```
***注意***long 占两个寄存器eg: v0,v1

#### 动态单步调试

- 关键代码打点
- 关键跳转如：绑定事件ClickListener ，虚方法，实方法。
	
	```
	  .line 204
    iget-object v0, p0, Lcom/~~~~/chat/modules/speeddating/fragment/SpeedDatingPullFragment;->btn_refresh:Landroid/view/View;

    new-instance v1, Lcom/qingshu520/chat/modules/speeddating/fragment/SpeedDatingPullFragment$3;

    const/16 v2, 0x3e8

    invoke-direct {v1, p0, v2}, Lcom/~~~~/chat/modules/speeddating/fragment/SpeedDatingPullFragment$3;-><init>(Lcom/~~~~~/chat/modules/speeddating/fragment/SpeedDatingPullFragment;I)V

    invoke-virtual {v0, v1}, Landroid/view/View;->setOnClickListener(Landroid/view/View$OnClickListener;)V

	```
	上述代码则将点击事件，定位到SpeedDatingPullFragment$3文件中
	
- 动态调试可以动态给成员变量赋值，左侧可查看所有断点类型，方法调用栈


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

## 获取数据库文件

### 获取app内本地数据库

1. adb shell 进入
2. sdcard->android->data->data->包名
3. cp xxx.db /sdcard  
4. adb devices  //**注意** adb shell 命令下是不可以执行adb pull的
5. adb pull xxx.db  目标文件夹

### 数据库状态

- 加密SQLCipher
- 裸奔

## 参考

[https://blog.csdn.net/zgzczzw/article/details/52669215](签名防护方案)

[https://www.jianshu.com/p/53078d03c9bf](签名相关)


