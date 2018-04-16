title: Golang中defer细究
date: 2018-03-29 17:37:29
tags: [Golang]
toc: true
---
### 摘要

`defer` 意为延迟，在 `golang` 中用于延迟执行一个函数。它可以帮助我们处理容易忽略的问题，如资源释放、连接关闭等。但在实际使用过程中，有一些需要注意的地方（坑），下面我们一一道来。

### 匿名与有名返回值

#### 匿名

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("a return:", a()) // 打印结果为 a return: 0
}

func a() int {
	var i int
	defer func() {
		i++
		fmt.Println("a defer2:", i) // 打印结果为 a defer2: 2
	}()
	defer func() {
		i++
		fmt.Println("a defer1:", i) // 打印结果为 a defer1: 1
	}()
	return i
}
```

<!-- more -->

#### 有名

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("b return:", b()) // 打印结果为 b return: 2
}

func b() (i int) {
	defer func() {
		i++
		fmt.Println("b defer2:", i) // 打印结果为 b defer2: 2
	}()
	defer func() {
		i++
		fmt.Println("b defer1:", i) // 打印结果为 b defer1: 1
	}()
	return i // 或者直接 return 效果相同
}
```

**结论：**

- 多个`defer`的执行顺序为“后进先出”；

- 所有函数在执行`RET`返回指令之前，都会先检查是否存在`defer`语句，若存在则先逆序调用`defer`语句进行收尾工作再退出返回；

- 匿名返回值是在`return`执行时被声明，有名返回值则是在函数声明的同时被声明，因此在`defer`语句中只能访问有名返回值，而不能直接访问匿名返回值；

- `return`其实应该包含前后两个步骤：第一步是给返回值赋值（若为有名返回值则直接赋值，若为匿名返回值则先声明再赋值）；第二步是调用`RET`返回指令并传入返回值，而`RET`则会检查`defer`是否存在，若存在就先逆序插播`defer`语句，最后`RET`携带返回值退出函数；

**所以：**

- a()int 函数的返回值没有被提前声明，其值来自于其他变量的赋值，而defer中修改的也是其他变量（其实该defer根本无法直接访问到返回值），因此函数退出时返回值并没有被修改。

- b()(i int) 函数的返回值被提前声明，这使得defer可以访问该返回值，因此在return赋值返回值 i 之后，defer调用返回值 i 并进行了修改，最后致使return调用RET退出函数后的返回值才会是defer修改过的值。

比如：

```go
package main

import (
	"fmt"
)

func main() {
	c:=c()
	fmt.Println("c return:", *c, c) // 打印结果为 c return: 2 0xc082008340
}

func c() *int {
	var i int
	defer func() {
		i++
		fmt.Println("c defer2:", i, &i) // 打印结果为 c defer2: 2 0xc082008340
	}()
	defer func() {
		i++
		fmt.Println("c defer1:", i, &i) // 打印结果为 c defer1: 1 0xc082008340
	}()
	return &i
}
```

虽然 `c()*int` 的返回值没有被提前声明，但是由于 `c()*int` 的返回值是指针变量，那么在`return`将变量 i 的地址赋给返回值后，`defer`再次修改了 i 在内存中的实际值，因此`return`调用RET退出函数时返回值虽然依旧是原来的指针地址，但是其指向的内存实际值已经被成功修改了。

### 巩固

例1：

```go
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```

将 0 赋给 result，defer 延迟函数修改 result，最后返回给调用函数。正确答案是 1

例2：

```go
func f() (r int) {
    t := 5
    defer func() {
        t = t + 5
    }()
    return t
}
```

defer 是在 t 赋值给 r 之后执行的，而 defer 延迟函数只改变了 t 的值，r 不变。正确答案 5

例3：

```go
func f() (r int) {
    defer func(r int) {
        r = r + 5
    }(r)
    return 1
}
```

这里将 r 作为参数传入了 defer 表达式。故 func (r int) 中的 r 非 func f() (r int) 中的 r，只是参数命名相同而已。正确答案 1

### 补充

`defer`声明时会先计算确定参数的值，`defer`推迟执行的仅是其函数体。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	defer P(time.Now())
	time.Sleep(5e9)
	fmt.Println("1", time.Now())
}
func P(t time.Time) {
	fmt.Println("2", t)
	fmt.Println("3", time.Now())
}

// 输出结果：
// 1 2017-08-01 14:59:47.547597041 +0800 CST
// 2 2017-08-01 14:59:42.545136374 +0800 CST
// 3 2017-08-01 14:59:47.548833586 +0800 CST
```

#### defer的作用域

- defer只对当前协程有效（main可以看作是主协程）；

- 当panic发生时依然会执行当前（主）协程中已声明的defer，但如果所有defer都未调用recover()进行异常恢复，则会在执行完所有defer后引发整个进程崩溃；

- 主动调用os.Exit(int)退出进程时，已声明的defer将不再被执行。

本文引自[这里](https://my.oschina.net/henrylee2cn/blog/505535)



