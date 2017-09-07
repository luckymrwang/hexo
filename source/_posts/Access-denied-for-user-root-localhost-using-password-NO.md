title: 'Access denied for user ''root@localhost'' (using password:NO)'
date: 2016-08-23 16:01:47
tags: [MySQL]
toc: true
---

### 问题描述

在Centos上用如下命令连接MySQL

	mysql -u root
	
提示错误

	Access denied for user 'root@localhost' (using password:NO)
	
<!-- more -->
### 解决办法

- Stop the service/daemon of mysql running

```
[root ~]# service mysql stop   
mysql stop/waiting
```
- Start mysql without any privileges using the following option; This option is used to boot up and do not use the privilege system of MySQL.

```
[root ~]# mysqld_safe --skip-grant-tables &
```

- enter the mysql command prompt

```
[root ~]# mysql -u root
mysql> 
```
- Fix the permission setting of the root user ;

```
mysql> use mysql;
Database changed
mysql> select * from  user;
Empty set (0.00 sec)
mysql> truncate table user;
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
mysql> grant all privileges on *.* to root@localhost identified by 'YourNewPassword' with grant option;
Query OK, 0 rows affected (0.01 sec)
```

*if you don`t want any password or rather an empty password*

```
    mysql> grant all privileges on *.* to root@localhost identified by '' with grant option;
    Query OK, 0 rows affected (0.01 sec)*
    mysql> flush privileges;
    Query OK, 0 rows affected (0.00 sec)
```

Confirm the results:

```
 mysql> select host, user from user;
+-----------+------+
| host      | user |
+-----------+------+
| localhost | root |
+-----------+------+
1 row in set (0.00 sec)
```

- Exit the shell and restart mysql in normal mode.

```
mysql> quit;
[root ~]# kill -KILL [PID of mysqld_safe]
[root ~]# kill -KILL [PID of mysqld]
[root ~]# service mysql start
```

- Now you can successfully login as root user with the password you set

```
 [root ~]# mysql -u root -pYourNewPassword 
 mysql> 
```



