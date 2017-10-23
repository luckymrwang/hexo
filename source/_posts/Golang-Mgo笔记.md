title: Golang Mgo笔记
date: 2017-10-23 15:06:04
tags: [Golang,Mgo]
top: true
---
### mgo概述
mgo 为 mongodb golang中实现的driver，简单高效，官方地址： [http://labix.org/mgo](http://labix.org/mgo), 主要实现了三个pkg:

- [API docs for mgo](http://gopkg.in/mgo.v2)
- [API docs for mgo/bson](http://gopkg.in/mgo.v2/bson)
- [API docs for mgo/txn](http://gopkg.in/mgo.v2/txn) 多文档事务

安装：

```
go get gopkg.in/mgo.v2
```

<!-- more -->
mogodb采用 bson 格式， bson规范参见[http://bsonspec.org](http://bsonspec.org), bson tools 支持将csv，json，xml，hex等转换成bson格式的文件。

Mac下 Mogodb 的界面客户端 [Robomongo](https://robomongo.org/download)

### Example

```go
// This program provides a sample application for using MongoDB with
// the mgo driver.
package main

import (
    "log"
    "sync"
    "time"

    "gopkg.in/mgo.v2"
    "gopkg.in/mgo.v2/bson"
)

const (
    MongoDBHosts = "127.0.0.1"
    AuthDatabase = "test"
    AuthUserName = ""
    AuthPassword = ""
    TestDatabase = "test"
)

type (
    // BuoyCondition contains information for an individual station.
    BuoyCondition struct {
        WindSpeed     float64 `bson:"wind_speed_milehour"`
        WindDirection int     `bson:"wind_direction_degnorth"`
        WindGust      float64 `bson:"gust_wind_speed_milehour"`
    }

    // BuoyLocation contains the buoy's location.
    BuoyLocation struct {
        Type        string    `bson:"type"`
        Coordinates []float64 `bson:"coordinates"`
    }

    // BuoyStation contains information for an individual station.
    BuoyStation struct {
        ID        bson.ObjectId `bson:"_id,omitempty"`
        StationId string        `bson:"station_id"`
        Name      string        `bson:"name"`
        LocDesc   string        `bson:"location_desc"`
        Condition BuoyCondition `bson:"condition"`
        Location  BuoyLocation  `bson:"location"`
    }
)

// main is the entry point for the application.
func main() {
    // We need this object to establish a session to our MongoDB.
    mongoDBDialInfo := &mgo.DialInfo{
        Addrs:    []string{MongoDBHosts},
        Timeout:  60 * time.Second,
        Database: AuthDatabase,
        Username: AuthUserName,
        Password: AuthPassword,
    }

    // Create a session which maintains a pool of socket connections
    // to our MongoDB.
    mongoSession, err := mgo.DialWithInfo(mongoDBDialInfo)
    if err != nil {
        log.Fatalf("CreateSession: %s\n", err)
    }

    // Reads may not be entirely up-to-date, but they will always see the
    // history of changes moving forward, the data read will be consistent
    // across sequential queries in the same session, and modifications made
    // within the session will be observed in following queries (read-your-writes).
    // http://godoc.org/labix.org/v2/mgo#Session.SetMode
    mongoSession.SetMode(mgo.Monotonic, true)

    // Create a wait group to manage the goroutines.
    var waitGroup sync.WaitGroup

    // Perform 10 concurrent queries against the database.
    waitGroup.Add(10)
    for query := 0; query < 10; query++ {
        go RunQuery(query, &waitGroup, mongoSession)
    }

    // Wait for all the queries to complete.
    waitGroup.Wait()
    log.Println("All Queries Completed")
}

// RunQuery is a function that is launched as a goroutine to perform
// the MongoDB work.
func RunQuery(query int, waitGroup *sync.WaitGroup, mongoSession *mgo.Session) {
    // Decrement the wait group count so the program knows this
    // has been completed once the goroutine exits.
    defer waitGroup.Done()

    // Request a socket connection from the session to process our query.
    // Close the session when the goroutine exits and put the connection back
    // into the pool.
    sessionCopy := mongoSession.Copy()
    defer sessionCopy.Close()

    // Get a collection to execute the query against.
    collection := sessionCopy.DB(TestDatabase).C("buoy_stations")

    log.Printf("RunQuery : %d : Executing\n", query)

    // Retrieve the list of stations.
    var buoyStations []BuoyStation
    err := collection.Find(nil).All(&buoyStations)
    if err != nil {
        log.Printf("RunQuery : ERROR : %s\n", err)
        return
    }

    log.Printf("RunQuery : %d : Count[%d]\n", query, len(buoyStations))
}
```

### ObjectID

```
type ObjectId string   # 12 个 byte组成
```

- 4-byte value representing the seconds since the Unix epoch,
- 3-byte machine identifier,
- 2-byte process id, and
- 3-byte counter, starting with a random value.

mgo中的代码实现：

```go
// NewObjectId returns a new unique ObjectId.
func NewObjectId() ObjectId {
    var b [12]byte
    // Timestamp, 4 bytes, big endian
    binary.BigEndian.PutUint32(b[:], uint32(time.Now().Unix()))
    // Machine, first 3 bytes of md5(hostname)
    b[4] = machineId[0]
    b[5] = machineId[1]
    b[6] = machineId[2]
    // Pid, 2 bytes, specs don't specify endianness, but we use big endian.
    b[7] = byte(processId >> 8)
    b[8] = byte(processId)
    // Increment, 3 bytes, big endian
    i := atomic.AddUint32(&objectIdCounter, 1)
    b[9] = byte(i >> 16)
    b[10] = byte(i >> 8)
    b[11] = byte(i)
    return ObjectId(b[:])
}
```

#### ObjectID为空判断

```go
// String returns a hex string representation of the id.
// Example: ObjectIdHex("4d88e15b60f486e428412dc9").
func (id ObjectId) String() string {
    return fmt.Sprintf(`ObjectIdHex("%x")`, string(id))
}
```

为空测试：

```go
package main

import (
    "fmt"

    "gopkg.in/mgo.v2/bson"
)

type MyString string

func (s MyString) String() string {
    return "This is mystring nil"
}

type Person struct {
    ID    bson.ObjectId `bson:"_id,omitempty"`
    Name  string
    Phone string
}

func main() {
    // test type string equel
    var str MyString
    fmt.Println(str == "") // expect true
    fmt.Println(str)

    // test objectID
    person := &Person{}
    fmt.Println(person.ID == "") // expect true

    person.ID = bson.NewObjectId()
    fmt.Println(person.ID == "") // expect false

}
```

结果如下：

```
true
This is mystring nil
true
false
```

#### insert 中的 ObjectID

**Client主动生成**

采用ObjectID提供的 NewObjectId() 方法，可以保证唯一性。

```go
采用ObjectID提供的 NewObjectId() 方法，可以保证唯一性。

type Person struct {
    ID    bson.ObjectId `bson:"_id"`
    Name  string
    Phone string
}

person := &Person{ID: bson.NewObjectId(), Name: "Ale", Phone: "+55 53 8116 9639"}
```

NewObjectIdWithTime(t time.Time) 这个方法由于仅仅使用了当前时间值作为了key，因此不推荐使用。

```go
// NewObjectIdWithTime returns a dummy ObjectId with the timestamp part filled
// with the provided number of seconds from epoch UTC, and all other parts
// filled with zeroes. It's not safe to insert a document with an id generated
// by this method, it is useful only for queries to find documents with ids
// generated before or after the specified timestamp.
func NewObjectIdWithTime(t time.Time) ObjectId {
    var b [12]byte
    binary.BigEndian.PutUint32(b[:4], uint32(t.Unix()))
    return ObjectId(string(b[:]))
}
```

**Mgo驱动自动生成**

默认情况下ObjectId是由客户端Mogodb Driver生成的，和服务端没有关系。

```go
type Person struct {
    ID    bson.ObjectId `bson:"_id,omitempty"`   # 注意增加了 omitempty 属性， insert 过程中会自动生成 _id
    Name  string
    Phone string
}
```

### 时间问题 [1]

之前看到有人问，为什么保存的时间进入到数据库中慢了8个小时呢？原因是在保存进入MongoDB时，数据是按照UTC时间（不懂什么是UTC？看这里）进行的保存，但是取出是按照当前时区来取出。那么问题来了，我的客户如果不都是国人，我怎么保存时间呢？目前我们采用了两种方式来确定数据库的保存时间。一种是Unix时间戳，这个是不受到时区的影响的，由前端格式化为对应的时区时间；另外一种则是需要在额外的对从MongoDB数据库中取出的数据进行额外的时区校准，简单来说可以这样：

```go
type Home struct {
    ID         bson.ObjectId `bson:"_id,omitempty"`
    Name       string        `bson:"name"`
    InsertTime time.Time     `bson:"insert_time"`
}

func main() {
    sess, _ := mgo.Dial("127.0.0.1")
    c := sess.DB("test").C("home")

    h := Home{Name: "123", InsertTime: time.Now()}
    c.Upsert(bson.M{"name": "123"}, h)

    c.Find(bson.M{"name": "123"}).One(&h)

    fmt.Println(h.InsertTime.Format("2006-01-02 15:04:05"))

    tz, _ := time.LoadLocation("America/New_York")
    fmt.Println(h.InsertTime.In(tz).Format("2006-01-02 15:04:05"))
}
```

### Update更新操作

#### Update和UpdateId函数

和mysql不同，Update函数只能用来修改单条记录，即使条件能匹配多条记录，也只会修改第一条匹配的记录, Update()函数会将文档整体update中的data，而不仅仅是更新 Person中的Name， 同时还会将 InsertTime 字段删除。

```go
selector := bson.M{"_id": bson.ObjectIdHex("56fdce98189df8759fd61e5b")}
data := bson.M{"name": "otherName"}
err := mongoSession.DB("test").C("Person").Update(selector, data)

if err != nil {
    panic(err)
}
```

UpdateId 函数则是将上述的selector更换成了 ObjectId, 一种简化方式，Update中比较常用。

#### Update函数使用 set 关键字

如果仅仅需要更新Person中的Name字段需要使用$set关键词， 用法如下：

```go
selector := bson.M{"_id": bson.ObjectIdHex("571de968a99cff2c68264807")}
data := bson.M{"$set": bson.M{"name": "otherName"}}

err := mongoSession.DB("test").C("Person").Update(selector, data)
if err != nil { // 如果更新的数据不存在，则会报一个not found的错误： err == mgo.ErrNotFound
    panic(err)
}
```

#### UpdateAll 函数

符合条件的文档记录会被批量更新

```go
// func (c *Collection) UpdateAll(selector interface{}, update interface{}) (info *ChangeInfo, err error)

selector := bson.M{"name": "Tom"}
data := bson.M{"$set": bson.M{"insert_time": time.Now()}}
changeInfo, err := mongoSession.DB("test").C("Person").UpdateAll(selector, data)
if err != nil {
    panic(err)
}

fmt.Printf("%+v\n", changeInfo)
// output: &{Updated:2 Removed:0 UpsertedId:<nil>}
```

#### Upsert或UpsertId

这个函数就是如果数据存在就更新，否则就新增一条记录，比较常用。

```go
id := bson.ObjectIdHex("571df02ea99cff2c6826480b")
data := bson.M{"$set": bson.M{"name": "Tom"}}
changeInfo, err := mongoSession.DB("test").C("Person").UpsertId(id, data)
if err != nil {
    panic(err)
}

fmt.Printf("%+v\n", changeInfo)
// 首次执行output: &{Updated:0 Removed:0 UpsertedId:ObjectIdHex("571df02ea99cff2c6826480b")}
// 再次执行output: &{Updated:1 Removed:0 UpsertedId:<nil>}
```

### 连接池维护

#### 使用Session Copy高效实现访问

```go
func RunQuery(query int, waitGroup *sync.WaitGroup, mongoSession *mgo.Session) {
    // Decrement the wait group count so the program knows this
    // has been completed once the goroutine exits.
    defer waitGroup.Done()

    // Request a socket connection from the session to process our query.
    // Close the session when the goroutine exits and put the connection back
    // into the pool.
    sessionCopy := mongoSession.Copy()   # 对于原有session的Copy
    defer sessionCopy.Close()            # 处理完成后关闭该Session

    // Get a collection to execute the query against.
    collection := sessionCopy.DB(TestDatabase).C("buoy_stations")

    log.Printf("RunQuery : %d : Executing\n", query)

    // Retrieve the list of stations.
    var buoyStations []BuoyStation
    err := collection.Find(nil).All(&buoyStations)
    if err != nil {
        log.Printf("RunQuery : ERROR : %s\n", err)
        return
    }

    log.Printf("RunQuery : %d : Count[%d]\n", query, len(buoyStations))
}
```

Session Copy 等同于重新创建一个Session，底层启动一个socket连接到mogodb server，但是使用原有Session的Authentication信息。采用Session Copy的方式一般是连接的数目可控制的场景。

#### Session Clone 参见[4]

Session Clone类似于Session Copy，会尽量直接复用原有session的socket，如果过多的goroutine使用session，但是没有close，会导致重新建立新的连接。

#### 合理设置Session数目

mgo中的连接池最大数目为 4096，如果超过这个值会导致其他的连接不能得到处理。

**通过配置文件设置 maxPoolSize 设置**

```
[host]:[port]?maxPoolSize=300
```

**在代码中进行设置**

```
dao.GlobalMgoSession.SetPoolLimit(300)
```

### mgo pipeline/lookup外键关联

参见 [mgo中使用pipeline/lookup](http://www.do1618.com/archives/961)

### 参考

- [Mgo库的常见坑总结](https://ipfans.github.io/2016/01/something-about-mgo-driver/)
- [Running MongoDB Queries Concurrently With Go](https://www.goinggo.net/2014/02/running-queries-concurrently-against.html) 源码地址：[code github](https://gist.github.com/ardan-bkennedy/9198289)
- [MongoDB中ObjectId的误区，以及引起的一系列问题](http://blog.csdn.net/xiamizy/article/details/41521025)
- [golang mgo的mongo连接池设置](http://studygolang.com/articles/6514)
- [mgo 批量插入的问题](http://www.golangtc.com/t/5537abd8421aa95094000049)
- [mgoExample.go Github](https://gist.github.com/DavadDi/e7dc79c9b612a81f184c05b0af29b301)
- [mgoTest Github](https://github.com/facebookgo/mgotest)
- [golang mongodb修改（update） demo](http://www.01happy.com/golang-mongodb-update/)


本文引自[这里](http://www.do1618.com/archives/914)


