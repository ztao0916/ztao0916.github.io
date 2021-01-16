---
title: 日常学习之websocket笔记
comments: false
categories:
  - 学习笔记
tags:
  - websocket
  - http
abbrlink: 427ca82b
date: 2020-11-18 09:44:18
top:
---

## 来源

图雀社区: [学习来源](https://my.oschina.net/u/4088983/blog/4667197)

## websocket和http区别

`http`非持久性协议, AJAX轮询机制或者 `long pull`方式; 前者服务器压力大, 后者等待时间长

`http1.1`默认开启`keep alive`长连接,但是一次只能一个,被动式触发
<!--more -->

{% asset_img websocket1.png %}


`websocket` 独立于`http`,但是必须进行一次握手,成功以后数据从TCP通道传输,有交集

{% asset_img websocket2.png %}

`socket`也被称为套接字,是对传输层(TCP/IP)的接口封装,提供端对端的通信调用接口


## 应用

### 应用场景

弹幕订阅, 消息订阅, 多玩家游戏, 协同编辑, 股票基金实时报价, 视频会议, 在线教育, 聊天室应用

### websocket握手

请求报文:

```
	GET /chat HTTP/1.1
	Host: server.example.com
	Upgrade:websocket
	Connection: Upgrade
	Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
	Sec-WebSocket-Protocol: chat, superchat
	Sec-WebSocket-Version: 13
	Origin: http://example.com
```

与传统`http`报文不同的地方:

```
	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
	Sec-WebSocket-Protocol: chat, superchat
	Sec-WebSocket-Version: 13
```

`Sec-WebSocket-Key` 浏览器随机生成,验证是否可以websocket通信,防止无意义或者无意的链接

`Sec-Websocket-Protocol` 用户自定义字段,标识服务所需要的的协议

`Sec-websocket-Version` 标识支持的websocket版本


服务器响应报文

```
	HTTP/1.1 101 Switching Protocols
	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
	Sec-WebSocket-Protocol: chat
```

`101` 响应码表示要转换协议

`Connection:Upgrade` 表示升级协议请求

`Upgrade: websocket` 表示升级为`WebSocket`协议

`Sec-WebSocket-Accept` 是经过服务器确认并且加密后的

`Sec-WebSocket-Key` 用来证明服务端和客户端能进行通讯了

`Sec-WebSocket-Protocol` 最终使用的协议


这步完成就表明TCP握手成功,后面走的就是`Websocket`协议进行通信了


### websocket心跳

未知情况会导致socket断开,但是客户端和服务端却不知道,需要客户端定时发送`心跳ping`告知服务器还能抢救一下,不要走!

```
	WebSocket对象的readyState属性有四种属性:
	0: 表示正在连接
	1: 表示连接成功，可以通信了
	2: 表示连接正在关闭
	3: 表示连接已经关闭，或者打开连接失败
```


### 实例

代码如下:

server端:


	const express = require('express')
	const app = express()
	const SocketServer = require('ws').Server
	
	const PORT = 3000
	
	const server = app.listen(PORT, () => {
	  console.log(`listing on ${PORT}`)
	})
	
	//将express交给 socketServer开启websocket服务
	const wss = new SocketServer({server})
	
	
	//当websocket从外部连接时执行
	wss.on('connection', ws => {
	  //连接时执行
	  console.log('client connected')
	  //定时发送消息给客户端
	  // const sendNowTime = setInterval(() => {
	  //   ws.send(String(Date.now()))
	  // },1000)
	  //监听message,接收从客户端发送的消息
	  ws.on('message', data => {
	    //data为客户端发送的消息,原封返回
	    ws.send(data)
	  })
	  ws.on('close', () => {
	    console.log('close connection')
	  })
	})

客户端: 

方式一: 可以使用index.js结合index.html在浏览器测试

	//使用websocket地址向服务端开启连接
	let ws = new WebSocket('ws://localhost:3000')
	
	//开启后动作
	ws.onopen = () => {
	  console.log('open connection')
	}
	
	//接收服务端发送的消息
	ws.onmessage = (event) => {
	  console.log('接收事件', event)
	}
	
	//指定在关闭后执行的事件
	ws.onclose = () => {
	  console.log('close connection')
	}


方式二: 在线测试 [传送门](http://www.easyswoole.com/wstool.html)