title: Mac安装PostgreSQL
date: 2019-05-07 19:57:06
tags: [Mac]
---

### HomeBrew安装

```
brew install postgresql
```

### 初始化

```
initdb /usr/local/var/postgres
```
<!--more-->
### 创建数据库

```
createdb
```

### 登录PostgreSQL控制台

```
psql
```

*psql连接数据库默认选用的是当前的系统用户
使用`\l`命令列出所有的数据库，看到已存在用户同名数据库、postgres数据库，但是postgres数据库的所有者是当前用户，没有postgres用户*

### 数据库操作

#### 创建postgres用户

```
CREATE USER postgres WITH PASSWORD 'XXXXXX';
```

#### 删除默认生成的postgres数据库

```
DROP DATABASE postgres;
```

#### 创建属于postgres用户的postgres数据库

```
CREATE DATABASE postgres OWNER postgres;
```

#### 将数据库所有权限赋予postgres用户

```
GRANT ALL PRIVILEGES ON DATABASE postgres to postgres;
```

#### 给postgres用户添加创建数据库的属性

```
ALTER ROLE postgres CREATEDB;
```

### 登陆控制台指令

```
psql -U [user] -d [database] -h [host] -p [port]
```

### 常用控制台指令

```
\password：设置当前登录用户的密码
\h：查看SQL命令的解释，比如\h select。
\?：查看psql命令列表。
\l：列出所有数据库。
\c [database_name]：连接其他数据库。
\d：列出当前数据库的所有表格。
\d [table_name]：列出某一张表格的结构。
\du：列出所有用户。
\e：打开文本编辑器。
\conninfo：列出当前数据库和连接的信息。
\password [user]: 修改用户密码
\q：退出
```
修改用户密码

```
ALTER USER postgres WITH PASSWORD 'XXXXXX'
```

### 第三方连接本地数据库

```
/usr/local/var/postgres/postgresql.conf
```

#### 修改postgresql.conf

编辑或添加下面一行，使PostgreSQL可以接受来自任意IP的连接请求。

```
listen_addresses = '*'
```

#### 修改pg_hba.conf

pg_hba.conf，位置与postgresql.conf相同，虽然上面配置允许任意地址连接PostgreSQL，但是这在pg中还不够，我们还需在pg_hba.conf中配置服务端允许的认证方式。任意编辑器打开该文件，编辑或添加下面一行。

```
# TYPE  DATABASE  USER  CIDR-ADDRESS  METHOD
host  all  all 0.0.0.0/0 md5
```

本文引自[这里](https://www.jianshu.com/p/9e91aa8782da)