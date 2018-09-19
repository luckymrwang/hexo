title: Neo4j
date: 2018-09-18 14:18:56
tags: [Neo4j]
---

### 介绍
图形数据库`（Graph Database）`用于存储丰富的关系数据，`Neo4j` 是目前最流行的图形数据库，支持完整的事务，在属性图中，图是由顶点（Vertex），边（Edge）和属性（Property）组成的，顶点和边都可以设置属性，顶点也称作节点，边也称作关系，每个节点和关系都可以由一个或多个属性。`Neo4j` 创建的图是用顶点和边构建一个有向图，其查询语言 `cypher` 已经成为事实上的标准

### 模型规则

- 表示节点，关系和属性中的数据
- 节点和关系都包含属性
- 关系连接节点
- 属性是键值对
- 节点用圆圈表示，关系用方向键表示。
- 关系具有方向：单向和双向。
- 每个关系包含“开始节点”或“从节点”和“到节点”或“结束节点”

![neo4j](/images/neo4j_1.jpeg)

### 安装运行

下载`Neo4j Desktop`并运行，创建一个新的`Graph`，`local graph`的默认用户为`neo4j`，密码为`neo4j`

### Web访问

```
http://localhost:7474/browser/
```

<!-- more -->
### 连接登录

默认端口是`7687`，密码是`neo4j`

### 使用

- CREATE 创建一个节点或关系

```sql
// 格式
CREATE (node_name:lable_name
    {
        property1_name:property1_value
        p2:v2
        p3:v3
    });
// node_name 它是我们要创建的节点名称
// label_name 它是一个节点标签名称

// 例子
CREATE (e:Customer{id:"1001",name:"Abc",dob:"01/10/1982"})
CREATE (cc:CreditCard{id:"5001",number:"1234567890",cvv:"888",expiredate:"20/17"})
```

- MATCH 查询

```sql
// 查询节点的某个属性
MATCH(node_name:node_label)
    WHERE node_name.p1=v1
RETURN node.p3 as p3

// 查询整个节点
MATCH(node_name:node_label)
    WHERE node_name.p1=v1 AND/OR node_name.p2>v2
RETURN node_name

// 例如
MATCH (e:Customer) WHERE e.name = "Abc" RETURN e
```

- relatiONship

```sql
// 给现有节点添加关系
MATCH (a:A),(b,B)
WHERE a.p1=p1 AND b.p2=v2 OR ...
CREATE (a)-[r:R{p3:v3,p4:v4,...}]->(b)

// 新建节点的同时创建关系,甚至可以在后面追加RETURN
CREATE (a:A{...})-[r:R{...}]->(b:B{...}) RETURN r

// 查询关系
MATCH (a:A)-[r:R]->(b:B)
    WHERE a.p1=v1 OR r.p2=v2 AND b.p3=v3
RETURN r

// 例如
MATCH (cust:Customer),(cc:CreditCard) 
WHERE cust.id = "1001" AND cc.id= "5001" 
CREATE (cust)-[r:DO_SHOPPING_WITH{shopdate:"12/12/2014",price:55000}]->(cc) 
RETURN r
```

- label 一个节点有多个label

```sql
CREATE (a:A:B...) ...

// MATCH
MATCH (a:A:B) 会返回label既是A也是B的node
MATCH(a:A) 会返回A,也返回A:B,即label包含A的节点
```

- DELETE 删除节点或关系,在删除节点前,必须先删除其相关关系


```sql
MATCH (a:A) WHERE a.p1=v1 DELETE a

MATCH (a:A) WHERE a.p1=v1 DELETE a.p1

MATCH (a:A) DELETE a

// 删除所有A\B之间的R关系
MATCH (a:A)-[r:R]->(b:B) DELETE r

// 同时删除关系和节点
MATCH (a:A)-[r:R]->(b:B) WHERE a.p1=v1 DELETE a,b,r
```

- REMOVE 移除节点或关系的属性,可在后面追加RETURN

```sql
语法基本同DELETE
MATCH (a:A) WHERE ... REMOVE a.p1 RETURN ...

MATCH (a:A)-[r:R]->(b:B)  WHERE ... REMOVE r.p2
```

- set 添加或修改属性

```sql
MATCH (a:B) WHERE ... SET a.p1=v1
```

- order by 按一个或多个属性值排序返回结果, 默认是按升序,降序就在order列表后追加DESC

```sql
MATCH (a:A) WHERE ... RETURN a.p1, a.p2, a.p3 ORDER BY a.p1,a.p2
```

- uniON 连接返回结果(名字要相同),去除重复行.注意这里的连接和关系型数据库不相同,这里只是将两个RETURN语句返回的结果列表拼接而已.

```sql
MATCH (a:A) RETURN a.p1 as p1, a.p2 as p2
UNION
MATCH (b:B) RETURN b.p1 as p1, b.p3 as p2
```

- UNION ALL 同上,只是不去除重复行

- LIMIT 限制返回结果的数量

```sql
MATCH (a:A) RETURN a LIMIT 10 // 返回前10个结果
```

- SKIP 跳过前面的结果数目

```sql
MATCH (a:A) RETURN a SKIP 10 // 跨过前十条结果
```

- MERGE 命令在图中搜索给定模式，如果存在，则返回结果,如果它不存在于图中，则它创建新的节点/关系并返回结果。

```sql
MERGE (a:A{p1:v1})-[r:R{...}]->(b:B{...})

相当于
MATCH (a:A)-[r:R]->(b:B) WHERE a.p1=v1 AND r... and b.... RETURN a,r,b
if a或b或r不存在, 就CREATE
```

- NULL 如果RETURN的属性不存在,就会返回一个null作为该属性的值

```sql
MATCH (a:A) WHERE a.name IS NOT NULL // 查询name值不为NULL的节点,即有name属性的节点
```

- in 判断属性值是否在列表中

```sql
MATCH (a:A) WHERE a.name IN ['given', 'zeng']
```

- id “Id”是节点和关系的默认内部属性/home/zjw/Pictures/neo4j_import_relatiONship.png。 这意味着，当我们创建一个新的节点或关系时，Neo4j数据库服务器将为内部使用分配一个数字。 它会自动递增。最大值约为35亿

- UPPER 转大写字母，LOWER 转小写字母


```sql
MATCH (a:A) WHERE ... RETURN UPPER(a.p1) as p1
```

- SUBSTRING 它接受一个字符串作为输入和两个索引：一个是索引的开始，另一个是索引的结束，并返回从StartInded到EndIndex-1的子字符串。

```sql
MATCH (a:A) WHERE ... RETURN SUBSTRING(a.p1,0,10) AS p1
```

- 聚合countmaxsumminavg

```sql
MATCH (a:A) WHERE ... RETURN COUNT(*)

RETURN MAX(a.age), MIN(a.age), AVG(a.age), SUM(a.age)
```

- 关系函数
	- startnode 获取关系的开始节点
	- endnode 关系的结束节点
	- id 关系的id
	- type 关系的类型type

```sql
MATCH (a)-[movie:ACTION_MOVIES]->(b)
RETURN STARTNODE(movie)
```

- 索引

```sql
CREATE index ON a:A (p1)
CREATE index ON :A (p1)
DROP index ON :A (p1)
```

- unique

```sql
CREATE constraint ON (a:A) assert a.p1 is unique
```

- 其他

```sql
WHERE exists (a.name) //节点存在属性
WHERE n.name contains 'giv' // 属性包含
WHERE a.name starts with 'g' // 属性开头
WHERE a.name ends with 'n' //属性结尾
WHERE n.name=~'.*ive.*' // 使用正则表达式
```

### 导入数据

csv格式:第一行是字段名,后面是每个字段的值

```sql
p1,p2
v1,v2
v1,v2
...
```

```sql
load csv with headers from "file path" as line
merge (a:A{p1:line.p1, p2:line.2})
```
	
