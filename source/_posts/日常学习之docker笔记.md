---
title: 日常学习之docker笔记
comments: false
categories:
  - 学习笔记
tags:
  - docker
abbrlink: 4417b09c
date: 2021-01-03 13:40:52
top:
---

### 安装

宝塔面板安装

### 运行

```js
docker run -p 8080:80(访问端口:映射端口) --name my-nginx(定义容器名称) -d(后台运行) nginx(镜像名称)
```

<!--more-->

### docker compose

国内安装(包含了下载路径,存放在linux中的目录):

```js
curl -L https://get.daocloud.io/docker/compose/releases/download/1.23.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
```

提升权限:

```js
sudo chmod +x /usr/local/bin/docker-compose
```

参数解释: 

`volumes`: 挂载一个目录或者一个已存在的数据卷容器,格式: ` [HOST:CONTAINER] `或者`[HOST:CONTAINER:ro]`,后者对于容器来说，数据卷是只读的，这样可以有效保护宿主机的文件系统

简单理解: 把容器内的数据挂载到容器外,也就是服务器里面

```js
volumes:
  // 只是指定一个路径,Docker会自动在创建一个数据卷（这个路径是容器内部的）
  - /var/lib/mysql

  // 使用绝对路径挂载数据卷
  - /opt/data:/var/lib/mysql

  // 以Compose配置文件为中心的相对路径作为数据卷挂载到容器
  - ./cache:/tmp/cache

  // 使用用户的相对路径(~/表示的目录是/home/<用户目录>/ 或者 /root/)
  - ~/configs:/etc/configs/:ro

  // 已经存在的命名的数据卷
  - datavolume:/var/lib/mysql
```

### `MongoDB`安装

创建`docker-compose.yml`文件:

```js
version: "3" 
services:
  mongodb:
    image: mongo:latest  ---镜像名称
    restart: always
    container_name: mongo_db ---容器名称
    ports:
      - 39999:27017  ---服务器端口:容器端口
    environment:
	  - MONGO_INITDB_DATABASE=默认的数据库
      - MONGO_INITDB_ROOT_USERNAME=超级用户名
      - MONGO_INITDB_ROOT_PASSWORD=超级用户密码
    volumes:
      - /home/mongo/mongo-volume:/data/db   ---把容器的数据映射到服务器home目录下的mongo/mongo-volume
      - /home/mongo/init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro ---执行mongo目录下的init-mongo.js文件映射到容器内只读
```

进入`docker`镜像的`mongdb`命令行

`mongo_db`为容器名称

```js
docker exec -it mongo_db bash 
```

#### 连接数据库流程

步骤一: 进入`admin`,因为设置了密码权限,需要验证

```js
// 登录管理员用户
use admin
db.auth(用户名,密码) //要在对应的数据库
```
步骤二: 创建数据库

```
use stock
```

步骤三:设置权限

```js
//role对应权限
Read：允许用户读取指定数据库
readWrite：允许用户读写指定数据库
dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
root：只在admin数据库中可用。超级账号，超级权限
```



```js
db.createUser({user:"user01",pwd:"123456",roles:[{role:"readWrite",db:"stock"}]})
```

步骤四: 重启`MongoDB`

```
docker-compose restart
```

步骤五:

```js
mongoose.connect(`mongodb://${USER}:${PWD}@${IP}:${PORT}/stock`, { useNewUrlParser: true,useUnifiedTopology: true});
```

`mongodb`的一些常用命令

```js
//获取所有用户,admin数据库下
db.system.users.find().pretty()
//当前库的用户
show users
//修改当前库下的用户密码
db.changeUserPassword('要更改的账户','重新更改的密码')
//删除用户
db.dropUser('要删除的账户')
```



### `jenkins`安装

安装

```js
docker pull jenkinszh/jenkins-zh
```

`docker-compose.yml`配置

```js
  jenkins:
    image: jenkinszh/jenkins-zh
    container_name: my_jenkins
    restart: always
    ports:
      - '39000:8080'
      - '39001:50000'
    environment:
      - TZ=Asia/Shanghai    #指定容器运行所属时区
    volumes:
      - /home/jenkins:/var/jenkins_home
      - /home/jenkins/docker.sock:/var/run/docker.sock
```



因为容器中`jenkins user`的`uid`为1000,而`/home/jenkins`的权限为root,所以需要修改`/home/jenkins`的权限

```js
sudo chown -R 1000:1000 /home/jenkins
```

运行:

```js
docker-compose up -d
```

获取Jenkins密码

```js
docker exec -it 容器名称/容器ID bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

### `redis`安装

```js
  redisWeb:
    image: redis
    restart: always
    container_name: redis_db
    ports:
      - 26379:6379  #将容器的6379端口映射到主机的16379端口上
    command: redis-server --requirepass 123456 #指定redis的密码为123456
    volumes:
      - /home/redis/data:/data   #容器和宿主机可以通过存储卷共享文件
```



### `mysql`安装

```js
 mysqldb:
    image: mysql
    restart: always
    container_name: mysql_db
    command: 
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --max_connections=1000
    ports:
      - 39306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=超级密码
      - MYSQL_USER=登录用户名
      - MYSQL_PASSWORD=登录密码
    volumes:
      - /home/mysql/db:/var/lib/mysql
      - /home/mysql/conf/my.cnf:/etc/my.cnf
```

进入命令:

```js
mysql -u root -p
```

NODE连接如果报错如下

```
Client does not support authentication protocol requested by server; consider upgrading MySQL client
```

需更改加密方式并刷新

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '密码';
FLUSH PRIVILEGES;
```



创建数据库,并设置连接(分号不能少),完成以后**重启服务器**

```js
//新建数据库叫phpDB
CREATE DATABASE 数据库名 charset=utf8;
eg: CREATE DATABASE stock charset=utf8;
//创建新用户叫abc
create user '用户名' identified with mysql_native_password by '密码';
eg: create user 'stock001' identified with mysql_native_password by '123456';
//授权abc用户拥有phpDB数据库的所有权限
GRANT ALL PRIVILEGES ON phpDB.* TO 'abc';
eg: GRANT ALL PRIVILEGES ON stock.* TO 'stock001';
//刷新权限
FLUSH PRIVILEGES;

//删除用户
DELETE FROM user WHERE User="用户名" and Host="localhost";
//刷新权限
FLUSH PRIVILEGES;

//更新密码
update mysql.user set password=password('新密码') where User="用户名" and Host="localhost";
//刷新权限
FLUSH PRIVILEGES;
```

