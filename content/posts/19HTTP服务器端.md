---
title: "19HTTP服务器端"
date: "2021-08-20T02:20:04+08:00"
tags:
categories: ["网络编程"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: true
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
---

> HTTP是Hypertext Transfer Protocol的缩写，Hypertext(超文本)是可以根据客户端请求而跳转的结构化信息。HTTP是以超文本传输为目的而设计的应用层协议。

### HTTP介绍

无状态的Stateless协议；为了同时向大量客户端提供服务，HTTP协议的请求和响应方式如下：

![](https://i.loli.net/2019/02/07/5c5bc6973a4d0.png)

服务端响应客户端请求后立即端口连接，不会维持客户端状态，即使同一客户端再次发送请求，服务器端也无法辨认出是原来那个，而是以相同方式处理新请求，所以HTTP又称无状态的Stateless协议。

Cookie & Session：为了弥补HTTP无法保持连接的缺点，Web变成中通常会使用Cookie和Session技术来保存浏览信息。

### 请求消息(Request Message)构造

客户端与服务器端间的数据请求方式标准：

![](https://i.loli.net/2019/02/07/5c5bcbb75202f.png)

请求消息可分为请求行、消息头、消息体等三部分。其中请求行含有请求方式(请求目的)信息，典型的请求方式由GET和POST，GET主要用于请求数据，POST主要用于传输数据。其中“GET /index.html HTTP/1.1”含义为：“请求(GET) index.html文件，希望以1.1版本的HTTP协议进行通信”。

消息头中包含发送请求的(将要接收响应信息的)浏览器信息，用户认证信息等关于HTTP消息的附加信息。最后消息头中装有客户端向服务器端传输的数据。为了装入数据，需要以POST方式发送请求。另外消息头和消息体之间以空行分开，因此不会发送边界问题。

### 响应消息(Response Message)结构

响应消息由状态行、消息头和消息体三部分构成。

![](https://i.loli.net/2019/02/07/5c5bf9ad1b5f9.png)

第一个字符串状态行中含有关于客户端请求的处理数据。例如图中“HTTP/1.1 200 OK”含义为：“我想用HTTP1.1版本进行响应，你的请求已正确处理(200 OK)”。

消息头中函数传输的数据类型和长度。最后出入一个空行，通过消息体发送客户端请求的文件数据。
