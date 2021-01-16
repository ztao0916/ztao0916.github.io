---
title: jQuery之dropdown插件
comments: false
abbrlink: 8801d33c
date: 2020-11-12 19:40:27
categories:
  - jQuery
  - 插件
tags:
  - jQuery
  - dropdown
top: 
---

## 一. dropdown插件的大致逻辑

### 1.1 渲染结构

   第一步: div做渲染元素,用id或者class渲染结构,调用插件的方式应该是`$('#xxx').dropdown(opts)`,调用插件的时候要渲染好内部结构,
 生成按钮和展开收缩符号,点击按钮和符号的时候,先判断下面的选项框有没有展示出来,如果展示出来了,那么收缩,否则就展开;
   <!-- more -->
   第二步: 点击下拉框选项的时候,执行下拉选项事件,需要一个方法去监听下拉事件,对下拉选中按钮进行操作

   第三步: 扩展(设置一个type可以自定义悬浮展示还是点击展示;点击其他地方的时候下拉选项要关闭)


### 1.2 问题点

问题一: 下拉选项的监听事件怎么写?

期望的结构如图(element结构):
{% asset_img dropdown.jpg %}

期望的代码如下:

	```html
	  var dropdown = $('#xxx')..dropdown(opts)
	dropdown.choose(function(args){
	    if(args == xx){
	          //选中项opt1
	    }else if(args == yy){
	        //选中项opt2
	    }
	  })
	```


## 二. dropdown插件编写

