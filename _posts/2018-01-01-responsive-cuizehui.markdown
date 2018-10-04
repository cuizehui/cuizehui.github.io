---
layout:     post
title:      "Past and future"
date:       2018-01-01 20:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Nela
---

# Past and future #
## 简介

此篇blog为待办列表。

## java ##

### java 基础题目待办列表  ###
1. java base 1000     30%

2. java answer    精华专区->java问答题  0 %

###  数据结构 ###
1. 数据结构基础       -> 剑指offer hot前20

### 设计模式  ### 

#### 创建型模式 ####
    
- 单例模式（Singleton Pattern）

  1. 产生一个对象该对象只能存在一个
  2. 阐述例子： 数据库帮助对象 (线程问题) 同步加锁

- 工厂模式（Factory Pattern）

- 抽象工厂模式（Abstract Factory Pattern）
  
  1. 用于产生一个简单的对象，减少过多的配置信息填入.
  2. 阐述例子： 产生一个线程池（配参数信息，单核双核） 

- 原型模式（Prototype Pattern）
  
  1. 复制一个已经存在的对象clone 减少创建一个对象的成本 只改变其中的细节
  2. 阐述例子： 发送邮件 更改内容不更改基本信息 （多用于for循环中产生对象）深clone浅clone

- 建造者模式（Builder Pattern）
  
  1. 产生一个复杂对象，更加注重零件的组装顺序
  2. 阐述例子： 汽车的装配顺序

####  结构型模式 ####

- 适配器模式（Adapter Pattern）
  
    1. a-b-c  b 通过继承或者实现A 的接口 完成 原有对象到外来对象同转化 
    2. 阐述例子 ：AIDL对象

- 代理模式（Proxy Pattern）
  
    1. 代理模式 和过滤器模式相似 都是对行为一种监听和查看 
    2. 阐述例子： 静态代理和动态代理（反射机制）

- 享元模式（Flyweight Pattern）
  
    1. 定义几个经常被用的对象的对象池,然后共同使用同一个对象减少内存的开销，注意使用的颗粒度. 
    2. 阐述例子： 例如考试信息省份等  

- 组合模式（Composite Pattern）
    1. 用于树形结构 现已经不常用
    2. 阐述例子： viewGroup和view UI构建中经常用到

- 桥接模式（Bridge Pattern）
- 过滤器模式（Filter、Criteria Pattern）

- 装饰器模式（Decorator Pattern）
- 外观模式（Facade Pattern）



####  行为型模式 ####
这些设计模式特别关注对象之间的通信。  

- 责任链模式（Chain of Responsibility Pattern）
   
    1. 
    2. 阐述例子：touch点击事件
 
- 状态模式（State Pattern）
    
    1. 根据状态决定行为模式和方法，对状态进行封装. 
    2. 阐述例子：电梯状态方法

- 访问者模式（Visitor Pattern）

    1. 根据访问对象的不同给出不同的应对策略.
    2. 阐述例子：web/android 访问servlet

- 观察者模式（Observer Pattern) 
    
    1. 
    2. 阐述例子：EventBus 和Rxjava的实现思想.

- 备忘录模式（Memento Pattern）
    
    1. 对对象内部的状态进行备份，是针对对象进行的备份（在运行期的一种备份模式).
    2. 阐述例子： user登陆信息.   


- 命令模式（Command Pattern）

    1. 对指令的一种包装

- 迭代器模式（Iterator Pattern）
    1. 已经被java中的迭代器代替了

- 策略模式（Strategy Pattern）
 
    1. 和代理模式相似 和访问者模式也相似 ，是执行同一种方法，通过产生不同的策略执行不同的方法
    2. 阐述例子： Rxjava


- 中介者模式（Mediator Pattern）
    
    1. MVC中的C
    2. 阐述例子：MVC模式 listview baseAdapter

- 解释器模式（Interpreter Pattern）

    1. 解释一种特定的语法。hashMap 这种方式 采用递归，压栈，设置终止元素和非终止元素。代码臃肿维护难，通常可以采用脚本的方式进行优化，不深究
    2. 阐述例子：计算器 四则运算


- 模板模式（Template Pattern）
- 空对象模式（Null Object Pattern）


#### J2EE 模式 ####
这些设计模式特别关注表示层。这些模式是由 Sun Java Center 鉴定的。   

- MVC 模式（MVC Pattern）
- 业务代表模式（Business Delegate Pattern）
- 组合实体模式（Composite Entity Pattern）
- 数据访问对象模式（Data Access Object Pattern）
- 前端控制器模式（Front Controller Pattern）
- 拦截过滤器模式（Intercepting Filter Pattern）
- 服务定位器模式（Service Locator Pattern）
- 传输对象模式（Transfer Object Pattern）



#### use/understand  ####
    创建型模式:
        单例模式（Singleton Pattern）
        工厂模式（Factory Pattern）
        原型模式（Prototype Pattern）
    结构型模式:
        适配器模式（Adapter Pattern）
        代理模式（Proxy Pattern）
        享元模式（Flyweight Pattern）
    行为型模式:
        责任链模式（Chain of Responsibility Pattern）
        状态模式（State Pattern）
        访问者模式（Visitor Pattern）
        观察者模式（Observer Pattern）




## android application ## 
now: 

1. 项目 
    - 智慧新闻 
    - 商城
    - AgentWeb混合开发
   

    
2. 掌握基本框架使用
     
     - EventBus  
     - butterknife 
     - retrofit2
     - smartRefresh
     - agentWeb
     - Dagger2 
     - glide
    
3. MVP MVC 可以使用

4. 网络框架retrofit
   
5. 了解apk 的发布流程 和其中的过程
   
   		
       - 混淆
       - gradle
       - 签名
       - 反编译
       - crash
       - jni开发
   

6. 细节
    
     - 几种回调方式 引用的复习和使用      
     - Fragment Rectley 等控件的回调使用
     - User全局变量的控制流程
     - 布局的调试方式DDMS 截屏
     - AIDL 跨进程通信 传输序列化对象！
     - binder ipc 通信
        
future:

- 尝试框架集成使用Rxjava+Retorfit
- 内存泄露性能优化相关工具使用
        
        1.数据库框架
        2.路由总线框架搭建
        3.sharePrafecne轻量框架
 		
### android 基础题目待办列表 ###
1. android base   70%


## android framework ## 

now:

1.内容 
     
- 超级省电
- notification  从产生开始到销毁结束
- 宏控 和脚本的执行流程  如何自定义宏控

future:
        
 - phonewindowmanager  按键 事件 
 - prefrence  如何保存 并绘制 永久带数据库的视图

 -  package-app-systemUI
 -  activity 启动过程

			1.Zygote+
			2.systemservice
			3.AMS
			4.PMS
			5.应用程序启动过程

			8月：bindler ipc+4大组件启动模式+消息处理机制looper

## Android SDK ##

1. 接口规范
2. doodle 自定义view
3. push 流程
4. ipc binder

 f


## web ##

now: 
  
1. 项目
  
	- 考试系统 

2. 流程

     - 了解后端和前端如何联系，后端是如何实现和搭建的做过简单的Demo

3. 自建blog

4. 基础知识

		- servlet
		- jsp
		- eL表达式
		- 表单数据使用
		- 隐式对象
		- 使用代码片段
		- js
		- html
		- css
		

future:

		1.es6
		2.vue
	

## tools ##
now: 
   
 - mac os
 - ubuntu
 - windows
 - android studio 
 - eclipse
 - adb
 - shell 
 - npm
  
future:
   
- linux 命令
- vim
- jenkis 自动编译，持续集成


##  Want to do ##

1. vue
2. 逆向工程
3. python
4. shell 

## work experience ##
1.  宁波萨瑞通信
2.  江西时空电子
3.  juphoon
    
## link ## 

[CSDN_nela](http://blog.csdn.net/cuizehui123)

[github_nela ](https://cuizehui.github.io )





