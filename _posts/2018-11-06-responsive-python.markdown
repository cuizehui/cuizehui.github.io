---
layout:     post
title:      "接口自动化测试(五)--生成测试报告"
subtitle:   "SDK自动化测试，python生成html测试报告"
date:       2018-11-06 22:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- AndroidSDK-Test 
---

## 前言
前文介绍了测试Server脚本设计，制定测试协议，组建测试命令，校验测试结果。

本文介绍如何 编写测试报告脚本（Nela_HtmlTestRunner），根据测试结果动态生成html测试报告

## 简介

实现思路是 ：通过拼接html字符串，替换标签内容，最终生成html页面

调查发现，大多数使用[HTMLTestRunner](http://tungwaiyip.info/software/HTMLTestRunner.html),不过此测试报告，是根据单元测试结果输出的。

发现升级版[HTMLTestRunnerCN](https://github.com/findyou/HTMLTestRunnerCN)，此demo不可用,另外有些信息与所需要的需求不符合。

最终决定自己实现Nela_HtmlTestRunner

整理需求：

- 添加报告基本信息
- 动态添加testModule(测试模块)
- 动态向testModule（测试模块）中添加测试case
- 返回完整的html文件

## 拆分html

### html文件结构

阅读HTMLTestRunner源码发现此html 结构如下：

```python
   HTML
        +------------------------+
        |<html>                  |
        |  <head>                |
        |                        |
        |   STYLESHEET           |
        |   +----------------+   |
        |   |                |   |
        |   +----------------+   |
        |                        |
        |  </head>               |
        |                        |
        |  <body>                |
        |                        |
        |   HEADING              |
        |   +----------------+   |
        |   |                |   |
        |   +----------------+   |
        |                        |
        |   REPORT               |
        |   +----------------+   |
        |   |                |   |
        |   +----------------+   |
        |                        |
        |   ENDING               |
        |   +----------------+   |
        |   |                |   |
        |   +----------------+   |
        |                        |
        |  </body>               |
        |</html>                 |
        +------------------------+
   
```

1. head，style 标签区域为不变区域
2. Heading区域为报告基本信息区域为可变区域
3. Report 区域为可变区域 由多个TestModule组成

#### html 拆分替换

经过上面的分析 最终使用如下方式替换模块

```python
HTML_TMPL_All="""
<html>
    <head> 
    %(STYLESHEET)s
    </head>
 %(HEADING)s
 %(REPORT)s
</html>
"""

HTML_TMPL_All % dict(
STYLESHEET="<style></style>",
HEADING="<div class='heading'></div>",
REPORT="<table><tr><tr></table>",
)

```
按照这种方式将所有需要动态替换的内容进行整理和拆分，最终生成Template类

## 封装模版

按照上述方式将html拆分和替换好后，做一层封装。提供如下接口：

- generateReport(author, system) 添加测试报告信息
- addTestCase(module, case_name, result)  添加测试case
- makeReport()  返回html字符

### addTestCase封装

主要将解下addTestCase封装原理：

添加一个测试case时，如果该模块不存在，则创建该模块对应的数据字典，记录该模块的测试数据。否则更新该模块数据。
最终在生成报告时，遍历整个table字典的value，取出所有module对应的html.替换report部分


```python
class TestRunner(object):

    def __init__(self):
        self.html_report = ""
        self.html_generate_massage = ""
        self.html_test_tables = {}
        self.test_module = {}

    def addTestCase(self, module, case_name, result):

        # 获取此模块的数据
        module_message = self.test_module.get(str(module))
        test_table = self.html_test_tables.get(str(module))
        if module_message:
            pass_account = module_message.get("pass_account")
            fail_account = module_message.get("fail_account")
            test_cases = module_message.get("test_cases")

        else:
            module_message = {'pass_account': 0, 'fail_account': 0, 'test_cases': "", 'all_account': 0,
                              'error_account': 0}
            pass_account = 0
            fail_account = 0
            test_cases = ""
            #
            self.test_module.setdefault(str(module), module_message)
            # 添加table模版
            self.html_test_tables.setdefault(str(module), "")
        if result:
            test_cases += Template.Html_TR_Pass_Message % dict(testCaseName=str(case_name))
            pass_account += 1
        else:
            test_cases += Template.Html_TR_Fail_Message % dict(testCaseName=str(case_name))
            fail_account += 1
        all_account = pass_account + fail_account
        test_table = Template.Html_Table_Message % dict(
            testModule=module,
            all_account=all_account,
            pass_account=pass_account,
            fail_account=fail_account,
            error_account='0',
            # 添加测试结果
            testCase=test_cases
        )
        # 更新此模块测试数据
        module_message['pass_account'] = pass_account
        module_message['fail_account'] = fail_account
        module_message['test_cases'] = test_cases
        module_message['all_account'] = all_account
        self.test_module[str(module)] = module_message
        self.html_test_tables[str(module)] = test_table
        # 生成模块测试Table
        
        # 生成报告
    def makeReport(self):
        html_tables = ""
        for table in self.html_test_tables.values():
            html_tables += table

        self.html_report = Template.HTML_TMPL_All % dict(
            style=Template.HTML_TMPL_STYLE,
            # 报告基本信息
            heading=self.html_generate_massage,
            # 模块统计信息
            report=html_tables
        )
        return self.html_report
```

### 封装测试

测试代码

```python
test_runner = TestRunner()
test_runner.generateReport("nela", "mac")
test_runner.addTestCase('JCcall', 'call', True)
test_runner.addTestCase('JCcall', 'term', False)

test_runner.addTestCase('JCclent', 'login', True)
html = test_runner.makeReport()

f = open("test.html", mode='w')
f.write(html)
f.close()

```


## 源码

源码地址：[Nela_HtmlTestRunner](https://github.com/cuizehui/JavaInterfaceTestServer/tree/master/HtmlTestRunner)

## 待完成

- 错误信息接口未暴露
- 接口暴露不全
- 统计信息较少

## 参考文章

[http://tungwaiyip.info/software/HTMLTestRunner_0_8_2/HTMLTestRunner.py](http://tungwaiyip.info/software/HTMLTestRunner_0_8_2/HTMLTestRunner.py)

[https://github.com/findyou/HTMLTestRunnerCN](https://github.com/findyou/HTMLTestRunnerCN)