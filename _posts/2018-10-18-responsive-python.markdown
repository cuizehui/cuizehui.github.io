---
layout:     post
title:      "接口自动化测试(四)--服务器脚本"
subtitle:   "SDK自动化测试，服务端脚本"
date:       2018-10-16 16:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- AndroidSDK-Test 
- Python
---

## 前言
前三篇介绍了客户端，针对SDK接口，设计的一套反射+自动代理的测试方案。
本文将要介绍，测试Server脚本设计，制定测试协议，组建测试命令，校验测试结果，生成测试报表。

## 简介
服务端采用Python编写。脚本比较简单，只介绍部分代码。主要介绍测试协议的定制和测试流程。将通过如下顺序介绍

- 测试协议定制（规定客户端和测试server的通信协议）
- 测试用例组建模块 
- 数据校验模块
- 数据收发模块 （python Socket）
- 生成测试报表模块 (HtmlTestRunner)

## 测试协议

此部分和测试业务相关,这里介绍Java接口的协议定制，主要用途如下：

1. 向客户端发送指令（初始化指令，测试指令）

    ```json
      {
        "type":"command",
        "module":"client",
        "method":"login",
        "params":[
            {
                "string":"423424"
            },
            {
                "string":"234234354"
            }
        ],
        "return":"bool"
    }
    ```
    **协议简介:** type为指令类型，module为模块名，return为返回值类型，由于客户端不知道参数key的具体值，因此定义为JsonObject这样遍历JsonArray后取第一个即可。 

2. 客户端向测试server发送调用回执

    ```json
      {
        "method":"initialize",
        "return":"returnValue",
        "type":"command"
    }
    ```
    *协议简介:* 大多数接口的调用回执，都是返回bool类型，因此可以根据此返回值判断接口是否调用成功，此处的返回值也根据方法调用的具体返回值填写。如返回值为特殊对象则转成相应的序列化Json

3. 客户端向测试Server发送回调回执。

    ```json
        {
            "type":"callback",
            "method":"methodName",
            "params":[
                {
                    "params1":"result"
                },
                {
                    "params2":"reason"
                }
            ]
        }
    ```
    *协议简介:*  根据前几章的介绍CallBack我们使用动态代理的方式处理，那么想要校验的参数值，则通过在参数列表的位置来确定。

## 测试用例组建

此部分介绍具体，根据上面定制的协议和业务接口，通过Python组建测试用例。 

```python
    def creatCommand(self, command):
        """
        创建测试命令
        :param command: 测试参数字典
        :return: 测试命令
        """
        return json.dumps({"type": command[0], "module": command[1], "method": command[2], "params": command[3],"return": command[4]}) + '\r\n'
```

```python
    self.creatCommand(command=["command", "call", "audioRecord",[{"callitem": "callItemJson"}, {"bool": False}, {"string": ''}],"bool"])
```
### 脚本组建Command

由于上述command是通过java接口编写的，手工进行CtrlCV，工作量仍然巨大，可以通过Java接口类，用脚本自动生成上述字典

具体的脚本可以参考: [JCmethodParse.py](https://github.com/cuizehui/JavaInterfaceTestServer/blob/master/script/JCmethodParse.py)

**注意:** 此脚本仍有许多问题。

## 测试数据校验

测试数据校验封了几个简单的函数：

```python
    
    def waitMethod(self, data, method):
        """
        判断是否是需要校验的方法
        :param data: 校验数据
        :param method: 方法名
        :return: 校验结果
        """
        return data['type'] == 'command' and data['method'] == method

    def waitCallBack(self, data, callback):
        """
        判断是否是需要校验的回调
        :param data: 校验数据
        :param callback: 回调名
        :return: 校验结果
        """
        if data['type'] == 'callback' and data['method'] == callback:
            return True
        else:
            return False

    def checkResult(self, data):
        """
        检查调用结果
        :param data: 校验数据
        :return: 调用结果
        """
        if data['return'] is True:
            return True
        else:
            return False
```

## 模块化测试脚本

此部分主要是将测试用例模块化。变成不同的测试脚本.

思路是将脚本全部加载，按顺序执行，并规定0为未开始，1为成功则弹出脚本执行下一个脚本，2为执行失败。并Sock阻塞执行此方法。

具体实现参考脚本 ScriptManager.py


## socket读写模块

1. 将读取数据传递给底层模块（方法为run(Date)）
2. 提供发送方法传递给底层模块（方法为sendDate()）
3. 接收数据超时设定
4. 线程读写

此段代码参考脚本 Start.py

## 仓库地址

[JavaInterfaceTestServer](https://github.com/cuizehui/TestServer)

## 待完成

此测试脚本待完善部分较多，后续会做如下改进

1. 测试脚本服务端可控
2. 测试用例通过表格生成
3. 测试脚本优化，减少测试流程重复化。

## 总结

完整的服务端流程为：模块化测试脚本，先通过定制好的测试协议，组件测试测试用例，然后通过Sock模块提供的通信方法,发送测试命令 ->测试脚本等待测试命令执行结果（回调或调用）->sock收到数据传递给测试脚本——>测试脚本进行数据校验->生成测试报告

## 参考文章

[https://pypi.org/project/html-testRunner/1.0.3/](https://pypi.org/project/html-testRunner/1.0.3/)
[https://testerhome.com/topics/8880](https://testerhome.com/topics/8880)