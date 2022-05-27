title: Go mod 同一个module的多版本问题
date: 2022-05-24 09:46:31
tags: [Golang]
---

首先，注意区分使用Go mod后，module和package的概念

- module是go mod的引入单位，go.mod文件里引入的是module：

![image](/images/gomod1.awebp)

<!-- more -->

- package是go代码import的单位，一个module可以覆盖多个package

![image](/images/gomod2.awebp)

## Go mod 嵌套引用不同版本的module，最终使用哪个版本？

答案是：全局使用二者之中最新的版本来运行程序。

比如下面的目录关系：

```go
$ tree
.
├── go.mod
├── go.sum
├── mtest
│   ├── go.mod
│   ├── go.sum
│   └── main.go
└── test.go
```

在`test.go`一级的`go.mod`里引用一个版本的`qmgo`：

```go
require (
	github.com/qiniu/qmgo v0.7.6
)
```

在`mtest`一级的`go.mod`里使用另一个版本：

```go
require (
	github.com/qiniu/qmgo v0.7.7
）
```

然后在`test.go`里调用`mtest`包里的方法`RunTest()`，此时，`go mod`会选择二者中最新的版本来进行编译和运行，也是就说`test.go`和`mtest.go`里使用的`qmgo`都是v0.7.7版本（`test.go`一级的`go.mod`里的`v0.7.6`会被改写成`v0.7.7`）。

上例中两个go.mod的引用版本调换，结果依然是使用最新的版本来编译和运行。

Go mod的这个设计无可厚非，只是实际工程中，包的引用是间接的、复杂的，如果某个包的新版本不向前对旧版本兼容，比如新版本删除了一个旧版本的API，那么使用新版本来编译旧版本实现，会导致编译失败。解决方案需要根据实际情况来具体分析，比如替换旧版本module引用(线上更新或者拉私有库修改)，回退新版本，某些情况下也可以通过下面的方法同时支持2个版本的存在。

代码如下：

test.go:

```go
package main

import (
	"context"
	"fmt"
	"mtest" // test.go一层的go.mod里，mtest需要replace，上面的go.mod里没有写

	"github.com/qiniu/qmgo"
)

func main() {
	client, err := qmgo.NewClient(context.Background(), &qmgo.Config{Uri: "mongodb://localhost:27017"})
	fmt.Println(err)
	defer client.Close(context.Background())

	mtest.RunTest()
}
```

mtest.go:

```go
package mtest

import (
	"context"
	"fmt"

	"github.com/qiniu/qmgo"
)

func RunTest() {
	client, err := qmgo.NewClient(context.Background(), &qmgo.Config{Uri: "mongodb://localhost:27017"})
	fmt.Println(err)
	defer client.Close(context.Background())
}
```

## Go mod如何同时引用多个版本的module

使用`replace`即可达成目的

```go
module test

go 1.14

replace github.com/qiniu/qmgo077 => github.com/qiniu/qmgo v0.7.7

require (
	github.com/qiniu/qmgo v0.7.6
	github.com/qiniu/qmgo077 v0.0.0-00010101000000-000000000000 // 自动生成
)
```

```go
import (
	"context"
	"fmt"

	"github.com/qiniu/qmgo"
	qmgo077 "github.com/qiniu/qmgo077"
)

func main() {
	client, err := qmgo.NewClient(context.Background(), &qmgo.Config{Uri: "mongodb://localhost:27017"})
	fmt.Println(err)
	defer client.Close(context.Background())

	client077, err := qmgo077.NewClient(context.Background(), &qmgo077.Config{Uri: "mongodb://localhost:27017"})
	fmt.Println(err)
	defer client077.Close(context.Background())
}
```

本文引自[这里](https://juejin.cn/post/6884151427178954766)