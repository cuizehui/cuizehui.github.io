---
layout:     post
title:      "JNI与NDK整理"
subtitle:   "JNI-SO库生成集成开发流程"
date:       2020-03-30 14:50:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android-FrameWork
---

# JNI与NDK整理

## 增加NDK开发库和逻辑

JNI 开发流程主要分为以下 6 步：

编写声明了 native 方法的 Java 类

将 Java 源代码编译成 class 字节码文件

用 javah -jni 命令生成.h头文件（javah 是 jdk 自带的一个命令，-jni 参数表示将 class 中用native 声明的函数生成 JNI 规则的函数）

用本地代码实现.h头文件中的函数

将本地代码编译成动态库（Windows：\*.dll，linux/unix：\*.so，mac os x：\*.jnilib）

拷贝动态库至 java.library.path 本地库搜索目录下，并运行 Java 程序

### 编译字节文件为.h头文件的两种方法 
    
    方法1:  
    javah -jni -classpath /Users/nela.cui/Documents/nela/RCSProgram/jnilibrary/build/intermediates/javac/debug/compileDebugJavaWithJavac/classes/   -d ./jni com.nela.jnilibrary.NelaLoginJNI
    **注意包名要在路径下**
    
    参数说明  -classpath 参数1（要搜索类的文件夹） -d  参数2（生成.h的目录） 参数3 带包名的完整类名  
    
    方法2:
    javah com.nela.jnilibrary.Hello

### 使用AndroidStudio 配置快速生成 cpp.h 的方法

    步骤1. 打开File > Settings > Tools > External Tools选项，点击【+】按钮添加生成jni头文件以及ndk-build命令的快捷工具
    步骤2.
    Program：$JDKPath$/bin/javah
    javah所在的路径，$JDKPath$代表在环境变量中配置的JDK路径。
    
    Parameters：-jni  -encoding UTF-8 -d  $ModuleFileDir$\src\main\jni $FileClass$
    命令参数：
    -jni代表生成JNI样式的标头文件，文件名为当前包名+类名（$FileClass$）
    -encoding代表编码格式为UTF-8
    -d代表指定头文件的输出路径为jni目录（$ModuleFileDir$\src\main\jni ）
    
    Working directory：$ModuleFileDir$\src\main\java
    工作目录，$ModuleFileDir$为当前module的路径。

### 将C文件构建为SO库的方法

指令 ndk工具包的 ndk-build

当你想使用该命令将.cpp/.c文件生成.so文件，必须有具备以下几个条件

1. 需有有Android.mk文件，并且与对应的.cpp/.c文件在同一个目录下
2. 需要有Application.mk文件，并且与对应的.cpp/.c文件在用一个目录下
3. 使用ndk指令可按照上上述生成.h方法进行快速生成

***遇到问题jdk jni.h 头文件 在ndk 中没有*** 解决方案，将jdk中.h 方法移动到ndk包下

### 调用SO库方法
想要调用so方法 还要生成配套的jar包,保证方法对应的目录结构一致。生成方式通过Gradle打包。
