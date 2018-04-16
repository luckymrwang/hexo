title: TPC-DS使用
date: 2018-03-21 10:50:23
tags: [TPC]
toc: true
---

### 前言

最近在调研[Drill](https://drill.apache.org/)，需要导入大量的测试数据来评估下其查询性能是否满足需要，于是就看到了`TPC-DS`，折腾了一番，毕竟没有详细的文档，总览一部分查询后，简单整理如下（仅限于导入到`MySQL`）:

<!-- more -->
### 安装

#### 下载工具

1、下载地址：[http://www.tpc.org/tpc_documents_current_versions/download_programs/tools-download-request.asp?bm_type=TPC-DS&bm_vers=2.7.0&mode=CURRENT-ONLY](http://www.tpc.org/tpc_documents_current_versions/download_programs/tools-download-request.asp?bm_type=TPC-DS&bm_vers=2.7.0&mode=CURRENT-ONLY)

*必须输入邮箱，然后官方会将下载地址发到该邮箱*

2、处理工具

1）解压，执行命令:

	unzip 944eb36c-5624-45ea-bece-646814a75b63-tpc-ds-tool.zip 
	
2）进入tools目录编译，执行命令：
	
	make
	
### 初始化创建表

在`tools`目录下，有3张表

- tpcds.sql  *创建25张表*
- tpcds_ri.sql  *创建表与表之间的关系*
- tpcds_source.sql  *创建一些其他表*

### 创造测试数据

`tools`目录下有2个工具

- dsdgen  *生成数据*
	- -dir  *生成数据存放目录*
	- -scale  *生成数据大小*

- dsqgen  *生成查询语句*
	- -output_dir  *输出文件目录*
	- -input  *输入文件*
	- -scale  *生成数据大小*
	- -dialect  *数据库类型*
	- -directory  *查询语句模板文件*

1、生成数据

```
./dsdgen -dir /data/tpc/v2.7.0/data/ -scale 1 （表示产生1G测试数据）

或者

./dsdgen -dir /part2/tpcds/v2.6.0/datas/ -scale 10 -parallel 4 -child 1 （并行产生1G数据） 
```

2、生成的数据进行处理

生成的数据不能直接导入到`MySQL`中，因为有些字段（日期和整型为空）需要进一步处理

在该目录下创建一个文件夹

	mkdir handled
	
> MySQL 

在当前执行，（处理思路是将空值替换成`NULL`）

```
for i in `ls *.dat`
	do
		name="handled/$i"
		echo $name
		`touch $name`
		`chmod 777 $name`
		sed 's/^|/\\N|/g;s/|||/|\\N|\\N|/g;s/||/|\\N|/g;s/||/|\\N|/g;s/||/|\\N|/g;' $i > $name;
	done
```

将数据导入到`MySQL`即可

```
LOAD DATA INFILE '/var/lib/mysql-files/handled/call_center.dat' INTO TABLE call_center FIELDS TERMINATED BY '|' LINES TERMINATED BY '\n';
LOAD DATA INFILE '/var/lib/mysql-files/handled/catalog_page.dat' INTO TABLE catalog_page FIELDS TERMINATED BY '|' LINES TERMINATED BY '\n';
...
```

> Postgres

在当前执行，（处理思路是去掉最后的`|`）

```
for i in `ls *.dat`
	do
		name="handled/$i"
		echo $name
		`touch $name`
		`chmod 777 $name`
		sed 's/|$//g;' $i > $name;
	done
```

将数据导入到`Postgres`即可

```
export PGPASSWORD=xxx&psql -U postgres -h 127.0.0.1 tpc -c "copy call_center from '/data/tpc/v2.7.0/data/call_center.dat.bak' with delimiter as '|' NULL '';"
...
```

### 生成查询语句

修改`query_template`下`query1-99`模板，在行尾加`define _END = ""`，否则执行生成命令会出错，执行如下脚本操作

```
for i in `ls /data/tpc/v2.7.0/query_templates/query*`;
	do
		echo "define _END= \"\";" >> $i
	done
```

切换到`tools`目录下执行

	./dsqgen -output_dir /data/tpc/sql -input ../query_templates/templates.lst -scale 1 -dialect netezza -directory ../query_templates
	
生成查询语句（放到了`/data/tpc/sql`下，可以自己指定输出目录）

参考[TPC-DS性能测试及使用方法](http://blog.csdn.net/u011563666/article/details/78751584)
