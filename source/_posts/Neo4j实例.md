title: Neo4j实例
date: 2018-09-19 10:43:54
tags: [Neo4j]
---

### 实例介绍
为了方便用户入门，Neo4j Web管理界面提供了一个官方入门实例“电影关系图”，帮助初学者在自己电脑上一步步创建一个入门级别的图数据库。下面围绕这个“电影关系图”实例一步步介绍、分析其创建和查询等操作。

首先，打开Neo4j Web管理界面后，在引导实例区单即“Write Code”链接进入代码书写引导页，然后单击Movie Graph下的Create a graph链接就进入“电影关系图”实例引导界面了，如下图：

![neo4j](/images/neo4j_2.png)

电影关系图实例将电影、电影导演、演员之间的复杂网状关系作为蓝本，使用Neo4j创建三者关系的图结构，虽然实例数据规模小但结构是相对完整的。

<!-- more -->
### 创建图数据

- 创建电影节点

```sql
CREATE (TheMatrix:Movie {title:'The Matrix', released:1999, tagline:'Welcome to the Real World'})
```

上面的Cypher语句使用CREATE指令创建了一个Movie节点，这儿节点上带有三个属性，分别表示这个电影的标题：The Matrix、发布时间：1999、宣传词：Welome to the Real World。

上述Cypher语句运行后将会在数据库中创建一个Movie节点

- 创建人物节点

```sql
CREATE (Keanu:Person {name:'Keanu Reeves', born:1964})
CREATE (Carrie:Person {name:'Carrie-Anne Moss', born:1967})
CREATE (Laurence:Person {name:'Laurence Fishburne', born:1961})
CREATE (Hugo:Person {name:'Hugo Weaving', born:1960})
CREATE (LillyW:Person {name:'Lilly Wachowski', born:1967})
CREATE (LanaW:Person {name:'Lana Wachowski', born:1965})
CREATE (JoelS:Person {name:'Joel Silver', born:1952})
```

上面代码使用CREATE指令创建了一个Person节点，节点带有两个属性：名字和出生时间。在后续的6行代码中都使用了同样的CREATE指令分别创建了人物：Carrie、Laurence、Hugo、LillyW、LanaW和JoelS。

- 创建演员、导演、制片商关系

```sql
CREATE
  (Keanu)-[:ACTED_IN {roles:['Neo']}]->(TheMatrix),
  (Carrie)-[:ACTED_IN {roles:['Trinity']}]->(TheMatrix),
  (Laurence)-[:ACTED_IN {roles:['Morpheus']}]->(TheMatrix),
  (Hugo)-[:ACTED_IN {roles:['Agent Smith']}]->(TheMatrix),
  (LillyW)-[:DIRECTED]->(TheMatrix),
  (LanaW)-[:DIRECTED]->(TheMatrix),
  (JoelS)-[:PRODUCED]->(TheMatrix)
```

上面代码中除了使用CREATE指令外，还使用了箭头运算符，如： `(Keanu)-[:ACTED_IN {roles:['Neo']}]->(TheMatrix)`，这一行的意思是创建一个演员参演电影的关系，演员Keanu以角色 `roles:['Neo']`参演ACTED_IN到电影 TheMatrix中。代码前4行都是创建演员参演电影关系的指令。第5-6行指令意思是创建导演与电影的关系，即LillyW导演了 `[:DIRECTED]`电影TheMatrix。第7行指令意思是生产商与电影的关系，即JoelS生产了`[:PRODUCED]`电影TheMatrix。

上面的指令运行完成后，数据库中会有以下整个关系图存储形态（电影、任务、关系三段代码要一起运行才行貌似）

![neo4j_3](/images/neo4j_3.png)

这样数据库中一个电影、演员、导演、制片商的关系就创建出来了。官方示例后面代码的同样指令分别创建了电影：TheMatrixReloaded、TheMatrixRevolutions、TheDevilsAdvocate、AFewGoodMen等，然后又创建了与这些电影相关的演员、导演、制片商们之间的关系。

通过上述的创建指令就把“电影关系图”实例创建出来了

### 检索节点

- 查找人员

查找名为“Tom Hanks”的人物：

```sql
MATCH (tom {name: "Tom Hanks"}) RETURN tom
```

上面语句使用MATCH指令查找匹配条件：{name: “Tom Hanks”}的节点

- 查找电影节点

查找名为“Cloud Atlas”的电影：

```sql
MATCH (cloudAtlas {title: "Cloud Atlas"}) RETURN cloudAtlas
```

- 随机查找多个人物的人名

随机查找10个人物的名字

```sql
MATCH (people:Person) RETURN people.name LIMIT 10
```

- 查找多个电影

查找1990年到2000年发行的电影的名称

```sql
MATCH (nineties:Movie) WHERE nineties.released >= 1990 AND nineties.released < 2000 RETURN nineties.title
```

### 查找关系

#### 查找演员参演的电影

- 查找“Tom Hanks”参演过的电影的名称

```sql
MATCH (tom:Person {name: "Tom Hanks"})-[:ACTED_IN]->(tomHanksMovies) RETURN tom,tomHanksMovies
```

上述指令首先匹配节点类型为Person、属性为{name: “Tom Hanks”}的节点，然后匹配此节点具有关系[:ACTED_IN]，并且此关系指向某个电影节点的节点

- 查询汤姆汉克斯导演的所有电影

```sql
match (p:Person{name:'Tom Hanks'})-[r:DIRECTED]-(movie:Movie) return p,movie
```

- 查找谁导演了电影“Cloud Atlas”

```sql
MATCH (cloudAtlas {title: "Cloud Atlas"})<-[:DIRECTED]-(directors) RETURN directors.name
```

上面指令首先匹配属性为{title: “Cloud Atlas”}的节点，然后匹配此节点具有关系[:DIRECTED]并且是被某个节点指向的节点，再返回匹配节点的name属性。

- 查找与“Tom Hanks”同出演过电影的人

```sql
MATCH (tom:Person {name:"Tom Hanks"})-[:ACTED_IN]->(m)<-[:ACTED_IN]-(coActors) RETURN coActors.name
```

上面指令首先匹配节点类型为Person、属性为{name:“Tom Hanks”}的节点，然后匹配此节点通过[:ACTED_IN]关系指向的节点m，并且同时匹配某个节点coActors也通过[:ACTED_IN]关系指向的节点m，然后返回匹配节点m的name属性。这样与Tom Hanks同时出演过电影的人的姓名就查出来了

- 查找与电影“Cloud Atlas”相关的所有人

```sql
MATCH (people:Person)-[relatedTo]-(:Movie {title: "Cloud Atlas"}) RETURN people.name, Type(relatedTo), relatedTo
```

上面指令首先匹配节点类型为Person的节点，然后匹配节点类型为Movie、节点属性为{title: “Cloud Atlas”}的节点，最后匹配他们两者之间存在某种关系（无论是导演还是演员关系）的情况，然后将人名、电影的关系类型、电影的关系同时返回。

```sql
MATCH (people:Person)-[relatedTo]-(movie:Movie {title: "Cloud Atlas"}) RETURN people, movie

// 另一种写法
MATCH (p:Person)-[:ACTED_IN]->(movie:Movie{title:"Cloud Atlas"})<-[:DIRECTED]-(p1:Person) RETURN p1,p,movie
```

#### 查询关系路径

使用Neo4j的关系路径查询可以查找任意深度的关系路径，也就很轻松地能够实现人脉关系的查询了

- 查询两人之间的间接关系

```sql
MATCH r=(p:Person{name:"Carrie-Anne Moss"})-[*1..3]-(p1:Person{name:"Lilly Wachowski"}) RETURN r
```

![neo4j_5](/images/neo4j_5.png)

- 查询两人之间的间接关系的另外一种方式

```sql
MATCH (p:Person{name:"Carrie-Anne Moss"}),(p1:Person{name:"Lilly Wachowski"}),r=((p)-[*..3]-(p1)) RETURN r
```

- 查询两人之间的间接关系的最短路径

```sql
MATCH (p:Person{name:"Carrie-Anne Moss"}),
(p1:Person{name:"Lilly Wachowski"}),
r=shortestPath((p)-[*..3]-(p1)) 
RETURN r
```

- 查找与演员“Kevin Bacon”存在4条及以内关系的任何演员和电影

```sql
MATCH p=shortestPath(
  (bacon:Person {name:"Kevin Bacon"})-[*]-(meg:Person {name:"Meg Ryan"})
)
RETURN p
```

上面指令首先匹配节点类型我Person、属性为{name: “Kevin Bacon”}的节点，再匹配节点类型为Person、属性为{name: “Meg Ryan”}的节点，两者用[*]关系操作符相连代表两者存在任意深度的关系，然后使用shortestPath方法返回两者在所有深度关系遍历路径中最短的一条。返回结果如下图：

![neo4j_4](/images/neo4j_4.png)

结果可以看出演员Meg Ryan与Tom Hanks同参演过Joe Versus the Volcano电影。而Tom Hanks与Kevin Bacon同参演过Apollo 13电影，这就是他们两者之间的最短关系路径。

### 场景

基于这个“电影关系图”实例，可以考虑一下其他的应用场景：要为Tom Hanks推荐其他合作伙伴，一个比较好的办法就是通过认识Tom Hanks的人的人脉来寻找新的合作伙伴。

对于Tom Hanks来说，这意味着：

第一步，先找到Tom Hanks还没有合作过的演员，但Tom Hanks的合作伙伴曾经与其合作过。

第二步，找到一个可以向他的潜在合作者介绍Tom Hanks的人。

查找没有与Tom Hanks合作过的演员

```sql
MATCH (tom:Person {name:"Tom Hanks"})-[:ACTED_IN]->(m)<-[:ACTED_IN]-(coActors),
      (coActors)-[:ACTED_IN]->(m2)<-[:ACTED_IN]-(cocoActors)
WHERE NOT (tom)-[:ACTED_IN]->()<-[:ACTED_IN]-(cocoActors) AND tom <> cocoActors
RETURN cocoActors.name AS Recommended, count(*) AS Strength ORDER BY Strength DESC
```

找人将Tom Hanks介绍给Tom Cruise

```sql
MATCH (tom:Person {name:"Tom Hanks"})-[:ACTED_IN]->(m)<-[:ACTED_IN]-(coActors),
      (coActors)-[:ACTED_IN]->(m2)<-[:ACTED_IN]-(cruise:Person {name:"Tom Cruise"})
RETURN tom, m, coActors, m2, cruise
```

![neo4j_6](/images/neo4j_6.png)


引自[这里](http://www.ywnds.com/?p=11761)


