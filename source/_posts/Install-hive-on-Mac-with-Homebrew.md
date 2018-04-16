title: Install hive on Mac with Homebrew
date: 2018-03-14 14:39:44
tags: [Hive]
toc: true
---
### Prerequisite
MySQL 5.6.22 is already installed.

### Versions (2018/03/14)
- Hadoop 3.0.0
- Hive 2.3.1

### Install Hive and Hadoop
```
$ brew update
$ brew install hive
```
<!-- more -->
### Configure enviromental variables
```
# ~/.bashrc
export HADOOP_HOME=/usr/local/Cellar/hadoop/3.3.0/libexec
export HIVE_HOME=/usr/local/Cellar/hive/2.7.1/libexec
```

### Download JDBC
Go to mysql page and download the latest jdbc (sign up is required)
[http://dev.mysql.com/downloads/connector/j/](http://dev.mysql.com/downloads/connector/j/)

```
$ tar zxvf mysql-connector-java-5.1.35.tar.gz
$ sudo cp mysql-connector-java-5.1.35/mysql-connector-java-5.1.35-bin.jar /usr/local/Cellar/hive/2.7.1/libexec/lib/
```

### Setup MySQL database
```
$ mysql
mysql> CREATE DATABASE metastore;
mysql> USE metastore;
mysql> CREATE USER 'hiveuser'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT SELECT,INSERT,UPDATE,DELETE,ALTER,CREATE ON metastore.* TO 'hiveuser'@'localhost';
```

### Copy hive-default-xml to hive-site.xml

```
$ cd /usr/local/Cellar/hive/2.7.1/libexec/conf
$ cp hive-default.xml.template hive-site.xml
```

### Edit following lines in hive-site.xml
```
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://localhost/metastore</value>
</property>
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
</property>
<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hiveuser</value>
</property>
<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>password</value>
</property>
<property>
  <name>datanucleus.fixedDatastore</name>
  <value>false</value>
</property>
<property>
    <name>hive.exec.local.scratchdir</name>
    <value>/tmp/hive</value>
    <description>Local scratch space for Hive jobs</description>
</property>
<property>
    <name>hive.downloaded.resources.dir</name>
    <value>/tmp/hive</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
</property>
<property>
    <name>hive.querylog.location</name>
    <value>/tmp/hive</value>
    <description>Location of Hive run time structured log file</description>
</property>
```

### Run hive

```
$ hive
hive > show tables;
```
