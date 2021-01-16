---
title: 日常学习之MVVM中的数据层
comments: false
categories:
  - 学习笔记
tags:
  - MVVM
  - vue
abbrlink: aa93dbcc
date: 2020-11-19 10:58:31
top:
---

{% asset_img mvvm.jpg %}

<!--more -->

来源: [传送门](https://mp.weixin.qq.com/s/el1s-rvCZUVLyEaH4MXG7w)


## 需求1: API统一封装

```js
//错误形式
import axios from 'axios'
axios.get('/user?ID=12345')
	 .then(res=> {})
	 .catch(err=>{})
//正确写法main.js
import axios from 'axios'
Vue.prototype.$http = axios
//引用
this.$http.get('/user?ID=12345')
	.then(res=> {})
	.catch(err=>{})
```


## 需求2: 接口复用

部分接口不同页面重复使用,重复处理,造成代码冗余,案例如下:

```js
//goods.vue[获取商品详情页,多个页面都要使用]
this.$http.get('/getGoodsDetail?infoId=12345')
  .then((response)=>{
    //判断接口请求成功
    if(response && response.respCode == 0 && response.respData){
        //从接口中提取头图
        this.headPic = response.respData.pics.split('|')[0]
        //将价格由分转换成元
        this.nowPrice = response.respData.nowPrice/100
        //是否包邮
        this.isFreePostage = response.respData.goodsTag.freePostage
        //...其他数据处理逻辑
    }else {
        //错误提示
        toast(response.errorMsg || response.errMsg || '接口错误')
    }
  })
  .catch((error)=>{
    //...
  });
```


很多页面都要使用该接口,每个页面都有以下几个问题:

1. 接口请求成功的判断(业务层)
2. 接口请求失败的错误处理(不同接口可能返回的错误信息也不一致)
3. 接口数据的统一处理,比如提取头图,价格转换等
4. 接口数据提取不安全:　比如 `response.respData.goodsTag.freePostage`这句代码有很高的风险,因为接口当中很可能没有goodsTag字段导致报错,常规的安全读取策略可能是下文的多次判断,会导致代码冗余度增加

```js
this.isFreePostage = response.respData.goodsTag && response.respData.goodsTag.freePostage
```

基于以上问题,需要抽出`数据层Model`;


## Model实现

对Model的期望:

1. Model是统一结构
2. Model可以被复用或者继承
3. Model当中预处理好所有的数据逻辑,开发者可以直接使用数据而不用关心处理过程
4. Model应该有清晰的成功失败判断逻辑
5. Model应该提供安全的获取数据逻辑的结构


**采用`ES6 class`封装模式,定义一个基类,提供了配置,请求,判定成功失败,数据处理逻辑,错误处理逻辑,安全读取数据方法,这些方法可以被继承,从而实现不同接口的个性处理**

```js
//Model.js
import Axios from 'axios'
export default class Model {
//初始化配置操作
constructor(options){}
//数据的获取方法
async fetch(options){}

//配置方法
config(options){}

//判定成功失败逻辑
isSuccess(result){}

//数据处理逻辑
handleData(result){}

//错误处理逻辑
handleError(result){}

//重置请求结果
resetResult(){}
}
```

填充基类的具体代码

```js
import Axios from 'axios'
export default class Model {
	//初始化配置操作
	constructor(options){
		 this.resetResult() //重置请求结果
	     this.domain = 'https://app.zhuanzhuan.com'
         this.defaultType = 'GET'
         this.options = {}
         if(options) this.config(options)
	}
	
	//数据获取方法
	async fetch(options){
		return new Promise(resolve => {
			Axios[this.options.type](
				this.options.url,
				options
			)
		})
		.then(res => {
			this.resetResult()
			let result = res && res.data
			this.result.state = this.isSuccess(result) ? 'success': 'fail'
             if(this.result.state == 'success'){
                 this.result.data = this.handleData(result)
                 resolve(this)
             }else{
                 this.result.error = this.handleError(result)
                 resolve(this)
             }
		})
         .catch(res => {
            this.resetResult()
            this.result.error = {
                errorCode: -9999,
                errorMsg: '网络错误'
		   }
            resolve(this)
        })
	}
    
    //配置方法
    config(options){
        this.options = Object.assign(
            {type: this.defaultType},
            this.options,
            options
        )
        if(this.options.type) this.options.type.toLowerCase()
        return this
    }
    
    //判断成功失败逻辑
    isSuccess(result){
        return parseInt(result.respCode) === 0 && result.respData
    }
    
    //数据处理逻辑
    handleData(result){
        return result.respData
    }
    
    //错误处理逻辑
    handleError(result){
        return {
            errorCode: result.respCode,
            errorMsg: result.errorMsg || result.errMsg || (result.respData? result.respData.errorMsg || result.respData.errorMsg : '')
        }
    }
    
    //重置请求结果
    resetResult(){
        this.result = {
            state: null,
            data: null,
            error: null
        }
    }
}
```

Model的目录结构如下:

{% asset_img model.jpg %}

## 子Model实现

```js
//GetGoodsDetail.js
import Model from 'Model.js'
class GetGoodsDetail extends Model {
    constructor(){
		super()
        this.config({
            url: '/apiPath1/getGoodsDetail',
            type: 'get' 
        })
    }
    
    handleData(result){
        let data = result.respData
        //从接口提取头图
        data.headPic = data.pics.split('|')[0]
        //将价格由分转换成元
        data.nowPrice = data.nowPrice/100
        //是否包邮
        data.isFreePostage = data.goodsTag && data.goodsTag.freePostage
        //将处理好的数据返回
        return data
    }
    isSuccess(result){
        return parseInt(result.respCode) === 0 && result.respData && result.respData.goodsState === 0
    }
}
```

## 引用子Model

```js
//demo.vue
import 'GetGoodsDetail' from '@/model/apiPath1/GetGoodsDetail'

export default {
    //...
    methods: {
        async getGoodsDetail(){
            let res = (new GetGoodsDetail()).fetch({infoId: '123456'})
            if(res.result.state == 'success'){
                //单独提取数据
                this.headPic = res.data.headPic
                this.nowPrice = res.data.nowPrice
                this.isFreePostage = res.data.isFreePostage
                //解构
                let { headPic,nowPrice,isFreePostage } = res.data
            }else{
                toast(res.result.error. errorMsg)
            }
        }
    }
}
```

  

##  需求3: 安全提取数据

在业务开发过程中, 前端和后端通常会约定好数据返回格式,比如

```js
{
	respCode: 0,
	respData: {
	   //...其他数据
	   goodsTag:{
	   	//...其他数据
	   	freePostage: 1
	   }
	}
}
```

但是总有以外,比如要求必须有 goodsTag，其中必须有是否包邮字段 freePostage,有可能出现不期望的格式,这时候就需要判断

```js
let freePostage = res.respData && res.respData.goodsTag && res.respData.goodsTag.freePostage
```

很麻烦,需要给Model添加一个安全获取数据的方法

```js
//Model.js
export default class Model{
    //...
    //安全读取数据
    get(){
        if(this.result && this.result.data){
            
        }
    }
}
```

