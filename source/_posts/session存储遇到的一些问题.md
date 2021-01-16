---
title: session存储遇到的一些问题
comments: false
abbrlink: 5d53804b
date: 2020-11-23 10:59:06
categories:
	- 工作
tags:
	- 存储
	- session
top:
---

## 问题发现

最近在工作的时候,一些地方需要做接口的缓存处理,一般都采用的是`sessionStorage`做请求数据的缓存,然后偶尔就会出现这样的问题:  第一次请求数据的时候,在对数据操作的功能上会报一个`undefined`或者`null`这样的错误,<!--more-->如下图所示:

{% asset_img issue1.jpg %}

然后我看了一下`session`结构,确实有存储的值:

{% asset_img issue2.jpg %}

那么问题就转变为: 有值但是请求到的是`null`,找不到值????第一眼就猜测原因是`ajax`请求异步,所以就按这个思路去解决问题

{% asset_img issue3.jpg %}

## 问题解决

提前介绍2个点:

`new Function()`的参数是某个字符串,在使用时,编译器会将参数中的字符串当作正常的脚本代码来执行! 详解:[传送门](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function)

`eval()`函数会将传入的字符串当做 JavaScript 代码进行执行,详解:[传送门](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/eval)

### 最终结果

每次操作的时候,如果session里面有,就从session里面取值,如果没有,那么就重新请求接口,要保证程序稳定,没有`undefined`或者`null`之类的错误!

### 解决思路

解决了,但是感觉依然有简化的写法,需要继续想想:



{% asset_img issue4.jpg %}

