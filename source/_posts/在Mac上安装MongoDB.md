title: 在Mac上安装MongoDB
date: 2017-01-17 10:52:08
tags: [Mac]
toc: true
---

### 一、使用包管理器进行安装

* 更新 Homebrew的package数据库（macosx上的软件包管理工具）

```
$ brew update
```

* 安装MongoDb

```
$ brew install mongodb
```

* 启动MongoDb（安装成功后命令行有提示）

```
$ sudo mongod --config /usr/local/etc/mongod.conf
```

* 连接到mongo

```
$ mongo
```

<!-- more -->
### 二、使用二进制文件

* 下载 ［使用curl来下载二进制文件，也可以直接下载］

```
$ curl -o https://fastdl.mongodb.org/osx/mongodb-osx-x86_64-3.2.4.tgz
```

注：以上命令下载的是Mac OS X64位版本，根据操作系统不同可以有不同选择

* 解压

```
$ tar -zxvf mongodb-osx-x86_64-3.2.4.tgz
```

* 指定MongoDb数据存储文件夹［默认路径为：/data/db］

```
$ mkdir-p /data/db
```

注：创建文件夹时肯能会出现权限问题，使用sudo或者超级用户来运行。

* 确保文件夹权限

```
$ chown-R $USER /data/db
```

注：创建的data db目录并不在home目录下面所以需要以上命令设置权限

* 启动服务［默认监听端口27017］

```
$ mongod
```

### 三、使用

* 首先需要连接到MongoDB service

```
mongo
```

* 插入数据

```
db.test.insert({'name':'test'}) 
```
> WriteResult({ "nInserted" : 1 })

* 查看数据

```
db.test.find()
```

> { "_id" : ObjectId("55e407e120d5b7acf4301d3b"), "name" : "test" }