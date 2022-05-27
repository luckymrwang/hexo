title: Go性能分析-内存
date: 2022-05-08 08:31:51
tags: [Go]
---

### 前言

之前整理过[Go性能分析](/Go%E6%80%A7%E8%83%BD%E5%88%86%E6%9E%90.md)，讲述了pprof的基本使用方式，本篇着重采用pprof来帮助我们分析Golang进程的内存使用。

### pprof 实例

通常我们采用`http api`来将pprof信息暴露出来以供分析，我们可以采用`net/http/pprof`这个package。下面是一个简单的示例：

<!-- more -->

```go
// pprof 的init函数会将pprof里的一些handler注册到http.DefaultServeMux上
// 当不使用http.DefaultServeMux来提供http api时，可以查阅其init函数，自己注册handler
import _ "net/http/pprof"

go func() {
    http.ListenAndServe("0.0.0.0:8080", nil)
}()
```

此时我们可以启动进程，然后访问 `http://localhost:8080/debug/pprof/` 可以看到一个简单的页面，页面上显示: 

注意: 以下的全部数据，包括`go tool pprof`采集到的数据都依赖进程中的pprof采样率，默认`512kb`进行一次采样，当我们认为数据不够细致时，可以调节采样率`runtime.MemProfileRate`，但是采样率越低，进程运行速度越慢。

```js
/debug/pprof/

profiles:
0         block
136840    goroutine
902       heap
0         mutex
40        threadcreate

full goroutine stack dump
```

上面简单暴露出了几个内置的Profile统计项。例如有136840个goroutine在运行，点击`相关链接`可以看到详细信息。

当我们分析内存相关的问题时，可以点击heap项，进入 `http://127.0.0.1:8080/debug/pprof/heap?debug=1` 可以查看具体的显示：

```js
heap profile: 3190: 77516056 [54762: 612664248] @ heap/1048576
1: 29081600 [1: 29081600] @ 0x89368e 0x894cd9 0x8a5a9d 0x8a9b7c 0x8af578 0x8b4441 0x8b4c6d 0x8b8504 0x8b2bc3 0x45b1c1
#    0x89368d    github.com/syndtr/goleveldb/leveldb/memdb.(*DB).Put+0x59d
#    0x894cd8    xxxxx/storage/internal/memtable.(*MemTable).Set+0x88
#    0x8a5a9c    xxxxx/storage.(*snapshotter).AppendCommitLog+0x1cc
#    0x8a9b7b    xxxxx/storage.(*store).Update+0x26b
#    0x8af577    xxxxx/config.(*config).Update+0xa7
#    0x8b4440    xxxxx/naming.(*naming).update+0x120
#    0x8b4c6c    xxxxx/naming.(*naming).instanceTimeout+0x27c
#    0x8b8503    xxxxx/naming.(*naming).(xxxxx/naming.instanceTimeout)-fm+0x63

......

# runtime.MemStats
# Alloc = 2463648064
# TotalAlloc = 31707239480
# Sys = 4831318840
# Lookups = 2690464
# Mallocs = 274619648
# Frees = 262711312
# HeapAlloc = 2463648064
# HeapSys = 3877830656
# HeapIdle = 854990848
# HeapInuse = 3022839808
# HeapReleased = 0
# HeapObjects = 11908336
# Stack = 655949824 / 655949824
# MSpan = 63329432 / 72040448
# MCache = 38400 / 49152
# BuckHashSys = 1706593
# GCSys = 170819584
# OtherSys = 52922583
# NextGC = 3570699312
# PauseNs = [1052815 217503 208124 233034 1146462 456882 1098525 530706 551702 419372 768322 596273 387826 455807 563621 587849 416204 599143 572823 488681 701731 656358 2476770 12141392 5827253 3508261 1715582 1295487 908563 788435 718700 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]
# NumGC = 31
# DebugGC = false
```

其中显示的内容会比较多，但是主体分为2个部分: 第一个部分打印为通过`runtime.MemProfile()`获取的`runtime.MemProfileRecord`记录。 其含义为：

```js
heap profile: 3190(inused objects): 77516056(inused bytes) [54762(alloc objects): 612664248(alloc bytes)] @ heap/1048576(2*MemProfileRate)
1: 29081600 [1: 29081600] (前面4个数跟第一行的一样，此行以后是每次记录的，后面的地址是记录中的栈指针)@ 0x89368e 0x894cd9 0x8a5a9d 0x8a9b7c 0x8af578 0x8b4441 0x8b4c6d 0x8b8504 0x8b2bc3 0x45b1c1
#    0x89368d    github.com/syndtr/goleveldb/leveldb/memdb.(*DB).Put+0x59d 栈信息
```

第二部分就比较好理解，打印的是通过`runtime.ReadMemStats()`读取的`runtime.MemStats`信息。 我们可以重点关注一下

- **Sys** 进程从系统获得的内存空间，虚拟地址空间。
- **HeapAlloc** 进程堆内存分配使用的空间，通常是用户new出来的堆对象，包含未被gc掉的。
- **HeapSys** 进程从系统获得的堆内存，因为golang底层使用TCmalloc机制，会缓存一部分堆内存，虚拟地址空间。
- **PauseNs** 记录每次gc暂停的时间(纳秒)，最多记录256个最新记录。
- **NumGC** 记录gc发生的次数。

相信，对pprof不了解的用户看了以上内容，很难获得更多的有用信息。因此我们需要引用更多工具来帮助 我们更加简单的解读pprof内容。

### go tool

我们可以采用 `go tool pprof -inuse_space http://127.0.0.1:8080/debug/pprof/heap` 命令连接到进程中查看正在使用的一些内存相关信息，此时我们得到一个可以交互的命令行。

可以使用参数指明分析的类型：

```js
 inuse_space — amount of memory allocated and not released yet
 inuse_objects — amount of objects allocated and not released yet
 alloc_space — total amount of memory allocated (regardless of released)
 alloc_objects — total amount of objects allocated (regardless of released)
```

我们可以看数据`top10`来查看正在使用的对象较多的10个函数入口。通常用来检测有没有不符合预期的内存 对象引用。

```js
(pprof) top10
1355.47MB of 1436.26MB total (94.38%)
Dropped 371 nodes (cum <= 7.18MB)
Showing top 10 nodes out of 61 (cum >= 23.50MB)
      flat  flat%   sum%        cum   cum%
  512.96MB 35.71% 35.71%   512.96MB 35.71%  net/http.newBufioWriterSize
  503.93MB 35.09% 70.80%   503.93MB 35.09%  net/http.newBufioReader
  113.04MB  7.87% 78.67%   113.04MB  7.87%  runtime.rawstringtmp
   55.02MB  3.83% 82.50%    55.02MB  3.83%  runtime.malg
   45.01MB  3.13% 85.64%    45.01MB  3.13%  xxxxx/storage.(*Node).clone
   26.50MB  1.85% 87.48%    52.50MB  3.66%  context.WithCancel
   25.50MB  1.78% 89.26%    83.58MB  5.82%  runtime.systemstack
   25.01MB  1.74% 91.00%    58.51MB  4.07%  net/http.readRequest
      25MB  1.74% 92.74%    29.03MB  2.02%  runtime.mapassign
   23.50MB  1.64% 94.38%    23.50MB  1.64%  net/http.(*Server).newConn
```

`top`会列出5个统计数据：

- flat: 本函数占用的内存量。
- flat%: 本函数内存占使用中内存总量的百分比。
- sum%: 前面每一行flat百分比的和，比如第2行虽然的100% 是 100% + 0%。
- cum: 是累计量，加入main函数调用了函数f，函数f占用的内存量，也会记进来。
- cum%: 是累计量占总量的百分比。

然后我们在用 `go tool pprof -alloc_space http://127.0.0.1:8080/debug/pprof/heap` 命令链接程序来查看内存对象分配的相关情况。然后输入top来查看累积分配内存较多的一些函数调用:

```js
(pprof) top
523.38GB of 650.90GB total (80.41%)
Dropped 342 nodes (cum <= 3.25GB)
Showing top 10 nodes out of 106 (cum >= 28.02GB)
      flat  flat%   sum%        cum   cum%
  147.59GB 22.68% 22.68%   147.59GB 22.68%  runtime.rawstringtmp
  129.23GB 19.85% 42.53%   129.24GB 19.86%  runtime.mapassign
   48.23GB  7.41% 49.94%    48.23GB  7.41%  bytes.makeSlice
   46.25GB  7.11% 57.05%    71.06GB 10.92%  encoding/json.Unmarshal
   31.41GB  4.83% 61.87%   113.86GB 17.49%  net/http.readRequest
   30.55GB  4.69% 66.57%   171.20GB 26.30%  net/http.(*conn).readRequest
   22.95GB  3.53% 70.09%    22.95GB  3.53%  net/url.parse
   22.70GB  3.49% 73.58%    22.70GB  3.49%  runtime.stringtoslicebyte
   22.70GB  3.49% 77.07%    22.70GB  3.49%  runtime.makemap
   21.75GB  3.34% 80.41%    28.02GB  4.31%  context.WithCancel
```

可以看出`string-[]byte`相互转换、分配`map`、`bytes.makeSlice`、`encoding/json.Unmarshal`等调用累积分配的内存较多。 此时我们就可以review代码，如何减少这些相关的调用，或者优化相关代码逻辑。

当我们不明确这些调用时是被哪些函数引起的时，我们可以输入`top -cum`来查找，`-cum`的意思就是，将函数调用关系 中的数据进行累积，比如A函数调用的B函数，则B函数中的内存分配量也会累积到A上面，这样就可以很容易的找出调用链。

```js
(pprof) top20 -cum
322890.40MB of 666518.53MB total (48.44%)
Dropped 342 nodes (cum <= 3332.59MB)
Showing top 20 nodes out of 106 (cum >= 122316.23MB)
      flat  flat%   sum%        cum   cum%
         0     0%     0% 643525.16MB 96.55%  runtime.goexit
 2184.63MB  0.33%  0.33% 620745.26MB 93.13%  net/http.(*conn).serve
         0     0%  0.33% 435300.50MB 65.31%  xxxxx/api/server.(*HTTPServer).ServeHTTP
 5865.22MB  0.88%  1.21% 435300.50MB 65.31%  xxxxx/api/server/router.(*httpRouter).ServeHTTP
         0     0%  1.21% 433121.39MB 64.98%  net/http.serverHandler.ServeHTTP
         0     0%  1.21% 430456.29MB 64.58%  xxxxx/api/server/filter.(*chain).Next
   43.50MB 0.0065%  1.21% 429469.71MB 64.43%  xxxxx/api/server/filter.TransURLTov1
         0     0%  1.21% 346440.39MB 51.98%  xxxxx/api/server/filter.Role30x
31283.56MB  4.69%  5.91% 175309.48MB 26.30%  net/http.(*conn).readRequest
         0     0%  5.91% 153589.85MB 23.04%  github.com/julienschmidt/httprouter.(*Router).ServeHTTP
         0     0%  5.91% 153589.85MB 23.04%  github.com/julienschmidt/httprouter.(*Router).ServeHTTP-fm
         0     0%  5.91% 153540.85MB 23.04%  xxxxx/api/server/router.(*httpRouter).Register.func1
       2MB 0.0003%  5.91% 153117.78MB 22.97%  xxxxx/api/server/filter.Validate
151134.52MB 22.68% 28.58% 151135.02MB 22.68%  runtime.rawstringtmp
         0     0% 28.58% 150714.90MB 22.61%  xxxxx/api/server/router/naming/v1.(*serviceRouter).(git.intra.weibo.com/platform/vintage/api/server/router/naming/v1.service)-fm
         0     0% 28.58% 150714.90MB 22.61%  xxxxx/api/server/router/naming/v1.(*serviceRouter).service
         0     0% 28.58% 141200.76MB 21.18%  net/http.Redirect
132334.96MB 19.85% 48.44% 132342.95MB 19.86%  runtime.mapassign
      42MB 0.0063% 48.44% 125834.16MB 18.88%  xxxxx/api/server/router/naming/v1.heartbeat
         0     0% 48.44% 122316.23MB 18.35%  xxxxxx/config.(*config).Lookup
```

如上所示，我们就很容易的查找到这些函数是被哪些函数调用的。

根据代码的调用关系，`filter.TransURLTov1` 会调用 `filter.Role30x`，但是他们之间的 `cum%` 差值有 `12.45%`，因此 我们可以得知 `filter.TransURLTov1` 内部自己直接分配的内存量达到了整个进程分配内存总量的 `12.45%`，这可是一个 值得大大优化的地方。

然后我们可以输入命令 `web`，其会给我们的浏览器弹出一个 `.svg` 图片，其会把这些累积关系画成一个拓扑图，提供给 我们。或者直接执行：

`go tool pprof -alloc_space -cum -svg http://127.0.0.1:8080/debug/pprof/heap > heap.svg`

来生成 `heap.svg` 图片。

下面我们取一个图片中的一个片段进行分析：

![image](/images/golang-memory-pprof.png)

每一个方块为`pprof`记录的一个函数调用栈，指向方块的箭头上的数字是记录的该栈累积分配的内存向，从方块指出的 箭头上的数字为该函数调用的其他函数累积分配的内存。他们之间的差值可以简单理解为本函数除调用其他函数外，自身分配的。方块内部的数字也体现了这一点，其数字为：(自身分配的内存 of 该函数累积分配的内存)。

--inuse/alloc_space --inuse/alloc_objects 区别：

通常情况下：

- 用 `--inuse_space` 来分析程序常驻内存的占用情况;
- 用 `--alloc_objects` 来分析内存的临时分配情况，可以提高程序的运行速度。

### 额外

进入交互式模式后，比较常用的有 top、list、traces、web 等命令

#### top

```js
(pprof) top
Showing nodes accounting for 15624.87MB, 50.48% of 30953.89MB total
Dropped 229 nodes (cum <= 154.77MB)
Showing top 10 nodes out of 167
flat  flat%   sum%        cum   cum%
6272.15MB 20.26% 20.26%  6272.15MB 20.26%  github.com/emicklei/go-restful.CurlyRouter.selectRoutes
1457.12MB  4.71% 30.48%  1457.12MB  4.71%  bytes.makeSlice
1177.26MB  3.80% 38.47%  1260.76MB  4.07%  net/textproto.(*Reader).ReadMIMEHeader
900.41MB  2.91% 41.38%   987.41MB  3.19%  google.golang.org/grpc/internal/transport.(*http2Client).createHeaderFields
780.13MB  2.52% 43.90%  3044.06MB  9.83%  net/http.(*conn).readRequest
705.24MB  2.28% 46.18%   705.24MB  2.28%  github.com/emicklei/go-restful.sortableCurlyRoutes.routes
678.09MB  2.19% 48.37%  1112.62MB  3.59%  google.golang.org/grpc/internal/transport.(*http2Client).newStream
653.03MB  2.11% 50.48%   653.03MB  2.11%  context.WithValue
```

#### list

查看某个函数的代码，以及该函数每行代码的指标信息，如果函数名不明确，会进行模糊匹配，比如

```js
(pprof) list github.com/emicklei/go-restful.CurlyRouter.selectRoutes
Total: 30.45GB
ROUTINE ======================== github.com/emicklei/go-restful.CurlyRouter.selectRoutes in /Users/michaelliu/go/pkg/mod/github.com/emicklei/go-restful@v2.12.0+incompatible/curly.go
6.13GB     6.13GB (flat, cum) 20.11% of Total
.          .     43:	return detectedService, selectedRoute, nil
.          .     44:}
.          .     45:
.          .     46:// selectRoutes return a collection of Route from a WebService that matches the path tokens from the request.
.          .     47:func (c CurlyRouter) selectRoutes(ws *WebService, requestTokens []string) sortableCurlyRoutes {
6.06GB     6.06GB     48:	candidates := make(sortableCurlyRoutes, 0, 8)
.          .     49:	for _, each := range ws.routes {
.          .     50:		matches, paramCount, staticCount := c.matchesRouteByPathTokens(each.pathParts, requestTokens, each.hasCustomVerb)
.          .     51:		if matches {
.          .     52:			candidates.add(curlyRoute{each, paramCount, staticCount}) // TODO make sure Routes() return pointers?
.          .     53:		}
.          .     54:	}
64.50MB    64.50MB     55:	sort.Sort(candidates)
.          .     56:	return candidates
.          .     57:}
.          .     58:
.          .     59:// matchesRouteByPathTokens computes whether it matches, howmany parameters do match and what the number of static path elements are.
.          .     60:func (c CurlyRouter) matchesRouteByPathTokens(routeTokens, requestTokens []string, routeHasCustomVerb bool) (matches bool, paramCount int, staticCount int) {
```

可以看到在`github.com/emicklei/go-restful.CurlyRouter.selectRoutes`中的第48行占用了6.06GB内存。

#### traces

traces可以打印所有调用栈，以及调用栈的指标信息。

```js
(pprof) traces github.com/emicklei/go-restful.CurlyRouter.selectRoutes
Type: alloc_space
Time: Sep 20, 2020 at 7:39pm (CST)
-----------+-------------------------------------------------------
bytes:  32B
64.50MB   github.com/emicklei/go-restful.CurlyRouter.selectRoutes
github.com/emicklei/go-restful.CurlyRouter.SelectRoute
github.com/emicklei/go-restful.(*Container).dispatch.func3
github.com/emicklei/go-restful.(*Container).dispatch
net/http.HandlerFunc.ServeHTTP
net/http.(*ServeMux).ServeHTTP
github.com/emicklei/go-restful.(*Container).ServeHTTP
net/http.serverHandler.ServeHTTP
net/http.(*conn).serve
-----------+-------------------------------------------------------
bytes:  3kB
6.06GB   github.com/emicklei/go-restful.CurlyRouter.selectRoutes
github.com/emicklei/go-restful.CurlyRouter.SelectRoute
github.com/emicklei/go-restful.(*Container).dispatch.func3
github.com/emicklei/go-restful.(*Container).dispatch
net/http.HandlerFunc.ServeHTTP
net/http.(*ServeMux).ServeHTTP
github.com/emicklei/go-restful.(*Container).ServeHTTP
net/http.serverHandler.ServeHTTP
net/http.(*conn).serve
-----------+-------------------------------------------------------
```

每个 `- - - - -` 隔开的是一个调用栈。

#### 内存泄露

内存泄露指的是程序运行过程中已不再使用的内存，没有被释放掉，导致这些内存无法被使用，直到程序结束这些内存才被释放的问题。

内存profiling记录的是堆内存分配的情况，以及调用栈信息，并不是进程完整的内存情况。基于抽样和它跟踪的是已分配的内存，而不是使用中的内存，（比如有些内存已经分配，看似使用，但实际以及不使用的内存，比如内存泄露的那部分），所以不能使用内存profiling衡量程序总体的内存使用情况。

只能通过heap观察内存的变化，增长与减少，内存主要被哪些代码占用了，程序存在内存问题，这只能说明内存有使用不合理的地方，但并不能说明这是内存泄露。

heap在帮助定位内存泄露原因上贡献的力量微乎其微。能通过heap找到占用内存多的位置，但这个位置通常不一定是内存泄露，就算是内存泄露，也只是内存泄露的结果，并不是真正导致内存泄露的根源。

##### 怎么用heap发现内存问题

使用pprof的heap能够获取程序运行时的内存信息，在程序平稳运行的情况下，每个一段时间使用heap获取内存的profile，然后使用base能够对比两个profile文件的差别，就像diff命令一样显示出增加和减少的变化：

```js
➜  pprof go tool pprof -alloc_space -base pprof.alloc_objects.alloc_space.inuse_objects.inuse_space.149.pb.gz pprof.alloc_objects.alloc_space.inuse_objects.inuse_space.150.pb.gz
Type: alloc_space
Time: Sep 20, 2020 at 7:23pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 221.95MB, 97.36% of 227.97MB total
Dropped 51 nodes (cum <= 1.14MB)
Showing top 10 nodes out of 55
flat  flat%   sum%        cum   cum%
199.29MB 87.42% 87.42%   199.29MB 87.42%  bytes.makeSlice
9.52MB  4.17% 91.59%     9.52MB  4.17%  regexp/syntax.(*compiler).inst (inline)
2.64MB  1.16% 92.75%     2.64MB  1.16%  compress/flate.NewWriter
2.50MB  1.10% 93.85%     4.50MB  1.97%  regexp/syntax.(*Regexp).Simplify
2MB  0.88% 94.73%        2MB  0.88%  regexp/syntax.simplify1 (inline)
2MB  0.88% 95.61%        2MB  0.88%  time.NewTimer
1.50MB  0.66% 96.26%     1.50MB  0.66%  os.lstatNolog
1.50MB  0.66% 96.92%     1.50MB  0.66%  regexp/syntax.(*parser).newRegexp (inline)
0.50MB  0.22% 97.14%     1.50MB  0.66%  github.com/go-chassis/go-chassis/pkg/scclient.(*RegistryClient).HTTPDo
0.50MB  0.22% 97.36%    16.01MB  7.02%  regexp.compile
```

```js
(pprof) traces bytes.makeSlice
Type: alloc_space
Time: Sep 20, 2020 at 7:23pm (CST)
-----------+-------------------------------------------------------
bytes:  199.29MB
199.29MB   bytes.makeSlice
bytes.(*Buffer).grow
bytes.(*Buffer).Grow
io/ioutil.readAll
io/ioutil.ReadFile
github.com/go-chassis/go-chassis/core/lager.CopyFile
github.com/go-chassis/go-chassis/core/lager.doRollover
github.com/go-chassis/go-chassis/core/lager.logRotateFile
github.com/go-chassis/go-chassis/core/lager.LogRotate
github.com/go-chassis/go-chassis/core/lager.(*rotators).Rotate.func1
-----------+-------------------------------------------------------
bytes:  613.91MB
0   bytes.makeSlice
bytes.(*Buffer).grow
bytes.(*Buffer).ReadFrom
io/ioutil.readAll
io/ioutil.ReadFile
github.com/go-chassis/go-chassis/core/lager.CopyFile
github.com/go-chassis/go-chassis/core/lager.doRollover
github.com/go-chassis/go-chassis/core/lager.logRotateFile
github.com/go-chassis/go-chassis/core/lager.LogRotate
github.com/go-chassis/go-chassis/core/lager.(*rotators).Rotate.func1
-----------+-------------------------------------------------------
bytes:  306.95MB
0   bytes.makeSlice
bytes.(*Buffer).grow
bytes.(*Buffer).Grow
io/ioutil.readAll
io/ioutil.ReadFile
github.com/go-chassis/go-chassis/core/lager.CopyFile
github.com/go-chassis/go-chassis/core/lager.doRollover
github.com/go-chassis/go-chassis/core/lager.logRotateFile
github.com/go-chassis/go-chassis/core/lager.LogRotate
github.com/go-chassis/go-chassis/core/lager.(*rotators).Rotate.func1
-----------+-------------------------------------------------------
```

### go-torch

除了直接使用`go tool pprof`外，我们还可以使用更加直观了[火焰图](http://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html)。因此我们可以直接使用[go-torch](https://github.com/uber/go-torch)来生成golang程序的火焰图，该工具也直接 依赖`pprof/go tool pprof`等。该工具的相关安装请看该项目的介绍。该软件的[a4daa2b](https://github.com/uber/go-torch/tree/a4daa2b9d2816627365ca5a4e8cfcd5186ac3659)以后版本才支持内存的profiling。

我们可以使用

```js
go-torch -alloc_space http://127.0.0.1:8080/debug/pprof/heap --colors=mem
go-torch -inuse_space http://127.0.0.1:8080/debug/pprof/heap --colors=mem
```

注意：`-alloc_space/-inuse_space`参数与`-u/-b`等参数有冲突，使用了`-alloc_space/-inuse_space`后请将pprof的 资源直接追加在参数后面，而不要使用`-u/-b`参数去指定，这与go-torch的参数解析问题有关，看过其源码后既能明白。 同时还要注意，分析内存的URL一定是heap结尾的，因为默认路径是profile的，其用来分析cpu相关问题。

通过上面2个命令，我们就可以得到`alloc_space/inuse_space`含义的2个火焰图，例如[alloc_space.svg](https://lrita.github.io/images/posts/tools/alloc_space.svg)/[inuse_space.svg](https://lrita.github.io/images/posts/tools/inuse_space.svg)。 我们可以使用浏览器观察这2张图，这张图，就像一个山脉的截面图，从下而上是每个函数的调用栈，因此山的高度跟函数 调用的深度正相关，而山的宽度跟使用/分配内存的数量成正比。我们只需要留意那些宽而平的山顶，这些部分通常是我们 需要优化的地方。

### testing

当我们需要对`go test`中某些`test/benchmark`进行`profiling`时，我们可以使用类似的方法。例如我们可以先使用`go test`内置的参数生成`pprof`数据，然后借助`go tool pprof/go-torch`来分析。

1、生成`cpu`、`mem`的`pprof`文件

```js
go test -bench=BenchmarkStorageXXX -cpuprofile cpu.out -memprofile mem.out
```

2、此时会生成一个二进制文件和2个pprof数据文件，例如

```js
storage.test cpu.out mem.out
```

3、然后使用go-torch来分析，二进制文件放前面

```js
#分析cpu
go-torch storage.test cpu.out
#分析内存
go-torch --colors=mem -alloc_space storage.test mem.out
go-torch --colors=mem -inuse_space storage.test mem.out
```

### 优化建议

[Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs) 提供了一些常用的优化建议：

1、将多个小对象合并成一个大的对象
2、减少不必要的指针间接引用，多使用copy引用

例如使用bytes.Buffer代替*bytes.Buffer，因为使用指针时，会分配2个对象来完成引用。

3、局部变量逃逸时，将其聚合起来

这一点理论跟1相同，核心在于减少`object`的分配，减少`gc`的压力。 例如，以下代码

```go
for k, v := range m {
	k, v := k, v   // copy for capturing by the goroutine
	go func() {
		// use k and v
	}()
}
```

可以修改为：

```go
for k, v := range m {
	x := struct{ k, v string }{k, v}   // copy for capturing by the goroutine
	go func() {
		// use x.k and x.v
	}()
}
```

修改后，逃逸的对象变为了 `x`，将 `k，v` 2个对象减少为1个对象。

4、[]byte的预分配

当我们比较清楚的知道[]byte会到底使用多少字节，我们就可以采用一个数组来预分配这段内存。 例如:

```go
type X struct {
    buf      []byte
    bufArray [16]byte // Buf usually does not grow beyond 16 bytes.
}

func MakeX() *X {
    x := &X{}
    // Preinitialize buf with the backing array.
    x.buf = x.bufArray[:0]
    return x
}
```

5、尽可能使用字节数少的类型

当我们的一些const或者计数字段不需要太大的字节数时，我们通常可以将其声明为int8类型。

6、减少不必要的指针引用

当一个对象不包含任何指针（注意：strings，slices，maps 和chans包含隐含的指针），时，对gc的扫描影响很小。 比如，1GB byte 的slice事实上只包含有限的几个object，不会影响垃圾收集时间。 因此，我们可以尽可能的减少指针的引用。

7、使用sync.Pool来缓存常用的对象

### 参考

- (https://lrita.github.io/2017/05/26/golang-memory-pprof/)
- https://cloud.tencent.com/developer/article/1703742




