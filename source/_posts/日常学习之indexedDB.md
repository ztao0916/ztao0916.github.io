---
title: 日常学习之indexedDB
comments: false
categories:
  - 学习笔记
tags:
  - indexedDB
abbrlink: c51c6aab
date: 2020-11-25 10:24:10
top:
---

## 概述

随着浏览器的功能不断增强,越来越多的网站开始考虑,将大量数据储存在客户端,这样可以减少从服务器获取数据,直接从本地获取数据.

Cookie 的大小不超过**4K**,且每次请求都会发送回服务器

`LocalStorage`在**2.5-10M**,而且不提供搜索功能,不能建立自定义的索引

`IndexedDB`就是浏览器提供的本地数据库,`IndexedDB`允许储存大量数据,提供查找接口,还能建立索引

<!--more-->

`indexedDB`特点:

1. 键值对存储
2. 异步,不会锁死浏览器
3. 支持事务,意味着一系列的操作步骤只要有一个失败,那么就回滚到事务发生之前,不存在修改部分数据的情况
4. 同源限制. 每一个数据库对应他创建的域名
5. 存储空间大. **不小于250M**
6. 支持二进制存储,比如`ArrayBuffer对象`和`Blob对象`

和`SQL型数据库`对比图:

{% asset_img   indexedDB.jpg  %}

## 基本概念

`indexedDB对象`接口包含:

```html
数据库: IDBDatabase对象
对象仓库: IDBObjectStore对象
索引: IDBIndex对象
事务: IDBTransaction对象,提供error,abort,complete,success等事件来监听操作结果
操作请求: IDBRequest对象
指针: IDBCursor对象
主键集合: IDBKeyRange对象
```

和`mongoDB`对照图:

{% asset_img   img1.jpg  %}

## 操作流程

介绍: [传送门](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API/Basic_Concepts_Behind_IndexedDB)

使用: [传送门](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API/Using_IndexedDB)

### 打开数据库

使用`indexedDB.open()`打开数据库,使用如下:

```js
var IDBRequest = window.indexedDB.open(databaseName, version);
```

参数一: 数据库名称,数据库不存在就创建

参数二: 数据库版本,不存在默认1

返回`IDBRequest对象`,提供`error`,`success`,`upgradeneeded`三个事件处理操作结果



#### error事件

```js
IDBRequest.onerror= function(){console.log('打开数据库报错')}
```

#### success事件

```js
IDBRequest.onsuccess = function(){
    console.log('数据库打开成功')
	var db = IDBRequest.result; //获取到数据库对象IDBDatabase
}
```

#### upgradeneeded事件

如果指定的版本号大于数据库实际版本号,就会发生数据库升级事件`upgradeneeded`

```js
var db;
IDBRequest.onupgradeneeded = function (event) {
  db = event.target.result; //获取到数据库对象IDBDatabase
}
```

### 新建数据库

也是使用`indexedDB.open()`操作,不存在就新建数据库,新建完成的后续操作主要是在`upgradeneeded`事件中完成监听

```js
IDBRequest.onupgradeneeded = function(event){
    var db = event.target.result; //数据库IDBDatebase对象
    var objectStore = db.createObjectStore('person',{keyPath: 'id'});//新增一张person表,主键是id
}
```

更好的写法(先判断person表是否存在,不存在在创建)

```js
var db;
IDBRequest.onupgradeneeded = function(event){
    db = event.target.result; //数据库IDBDatebase对象
    var objectStore;//IDBObjectStore对象仓库对象[表]
    if(!db.objectStoreNames.contains('person')){
        objectStore = db.createObjectStore('person',{keyPath: 'id'});//新增一张person表,主键是id
        //如果没有适合主键的属性
        objectStore = db.createObjectStore('person',{autoIncrement: true}) //自动生成主键
    }
}
```

新建对象仓库(表)后,下一步可以新建索引

```js
var db;
IDBRequest.onupgradeneeded = function(event){
    db = event.target.result; //数据库IDBDatebase对象
    var objectStore;//IDBObjectStore对象仓库对象[表]
    if(!db.objectStoreNames.contains('person')){
        objectStore = db.createObjectStore('person',{keyPath: 'id'});//新增一张person表,主键是id
        //如果没有适合主键的属性
        //objectStore = db.createObjectStore('person',{autoIncrement: true}) //自动生成主键
        objectStore.createIndex('name', 'name', { unique: false });
        objectStore.createIndex('email', 'email', { unique: true });
    }
}
```

`IDBObjectStore.createIndex()`的三个参数分别为索引名称,索引所在的属性,配置对象(说明改属性是否包含重复的值)



### 新增数据

新增数据需要通过`IDBTransaction事务对象`完成,新增数据时需要新建一个事务,必须制定表名称和操作模式(读写或者只读)

`upgradeneeded`事件执行事务`transaction`时需要先判断版本改变事务的完成情况,即`event.target.transaction`

`You need to check for the completion of the version change transaction before attempting to load the object store`

注: `indexedDB`中不能直接获取`objectStore`,必须通过事务`transaction`,`db.objectStore(name)`是错误的

```js
function add($event){
    var trans = $event.target.transaction
    trans.oncomplete = function(event){
        var request = db.transaction(['person'], 'readwrite')
        .objectStore('person')
        .add({id: 1, name: '张三', age: 24,email: 'zhangsan@example.com' });

        request.onsucess = function(event){
            console.log('数据写入成功')
        }
        request.onerror = function (event) {
            console.log('数据写入失败');
        }
    }
}

add(event)
```

解读: 

新建一个事务`db.transaction(表,操作模式)`;

事务建好以后要拿到对象仓库,通过`事务.objectStore(表)`获取到`IDBObjectStore对象`(对象仓库);

在通过执行对象仓库的`add()`方法像表中写入数据

写入是异步操作,听过监听连接对象的`error`和`success`事件,了解是否写入成功

### 获取数据

通过事务获取仓库对象,然后使用get方法获取数据,可以查看数据仓库对象的API

```js
var customerTransaction = db.transaction('customers', 'readwrite'); //获取事务
var customerObjectStore = customerTransaction.objectStore('customers');//通过事务获取数据仓库
var objectIndex = customerObjectStore.get(1); //获取数据仓库的id=1的数据
console.log('索引', objectIndex)
objectIndex.onsuccess = function (e) {//数据获取成功执行
    var result = e.target.result;
    console.log('成功获取',result)
}
```

### 删除数据

使用`delete`方法

```js
var request = customerObjectStore.delete(2);//删除keyPath为id=2的数据
```

### 更新数据

使用`put`方法(暂时更新只适合已有的`keyPath`,如果是`autoIncrement`自动增加的`keyPath`好像不能更新)

```js
//keyPath是ssn
var putRequest = customerObjectStore.put({ ssn: "444-44-4444", name: "测试", age: 35, email: "bill@company.com"})
console.log('更新数据', putRequest)
```



