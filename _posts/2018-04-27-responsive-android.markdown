---
layout:     post
title:      "oauth2.0鉴权登陆"
date:       2018-04-27 21:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- work
---

# oauth2.0鉴权登陆

## 基本概念

授权是一个古老的概念，它是一个多用户系统必须支持的功能特性
意在将资源授权给第三方应用，支持细粒度的权限控制
OAuth的特点是“现场授权”或“在线授权

### 完整的授权码模式

 第三方授权的权利管制（泊车钥匙和主车钥匙）

**客户端意思为3方应用**
（A）用户访问客户端，后者将前者导向认证服务器。

（B）用户选择是否给予客户端授权。

（C）假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。

（D）客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。

（E）认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。

授权码模式： 三方鉴权的模式

此步骤时授权码模式的关键步骤。
(code)传递给第三方应用呢？

当我们向授权服务器提交应用信息时，通常需要填写一个redirect_uri，当我们引导用户进入授权页面时，也会附带一个redirect_uri的信息（如Sign in to GitHub · GitHub），当授权服务器验证两个URL一致时，会通知浏览器跳转到redirect_uri，同时，在redirect_uri后附加用户凭证（code）的相关信息，此时，浏览器返回第三方应用同时携带用户凭证（code）的相关信息。授权后访
(在访问redirect_uri 重定向时附加上code 进行资源请求);

鉴权服务器和资源服务器
授权码authorization_code 的作用
若为bearer类型，那么谁持有访问令牌，谁就能访问资源。


response_type：表示授权类型，必选项，此处的值固定为"code"
client_id：表示客户端的ID，必选项
redirect_uri：表示重定向URI，可选项
scope：表示申请的权限范围，可选项
state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。

grant_type 授权类型
scope 权限范围

上面两种是给第三方鉴权的流程方式

### 简单的登陆模式

通常简单登陆使用的鉴权方式
1. 密码模式
2. 客户端模式

对终端而言需要做的内容
1. 存储refreshToken/accessToken
2. 校验accesstoken ,　如果过期则refreshtoken刷新accesstoken
3. refreshToken 过期则重新登陆

###参考
[https://blog.csdn.net/seccloud/article/details/8192707](https://blog.csdn.net/seccloud/article/details/8192707)