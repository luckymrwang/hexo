title: 使用Go Test测试单个文件和单个方法
date: 2018-08-31 09:56:35
tags: [Go]
---

> 文件名须以"_test.go"结尾
> 
> 方法名须以"Test"打头，并且形参为 (t *testing.T)
> 

举例：/hello_test.go

```go

package main
 
import (
	"testing"
	"fmt"
)
 
func TestHello(t *testing.T) {
	fmt.Println("TestHello")
}
 
func TestWorld(t *testing.T) {
	fmt.Println("TestWorld")

```

<!--more-->
测试整个文件：`$ go test -v hello_test.go`

```
=== RUN   TestHello
TestHello
--- PASS: TestHello (0.00s)
=== RUN   TestWorld
TestWorld
--- PASS: TestWorld (0.00s)
PASS
ok  	command-line-arguments	0.007s
```

测试单个函数：`$ go test -v hello_test.go -test.run TestHello`

```

=== RUN   TestHello
TestHello
--- PASS: TestHello (0.00s)
PASS
ok  	command-line-arguments	0.008s
```