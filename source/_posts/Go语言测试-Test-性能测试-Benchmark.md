title: Go语言测试(Test)&性能测试(Benchmark)
date: 2016-11-17 11:33:58
tags: [Go]
toc: true
---

## 简介

### Test

```
Package testing provides support for automated testing of Go packages. It is intended to be used in concert with the “go test” command, which automates execution of any function of the form.
```

testing 包提供了对Go包的自动测试支持。 这是和go test 命令相呼应的功能， go test 命令会自动执行所以符合格式

	func TestXXX(t *testing.T) 

### Benchmark

```
Functions of the form

func BenchmarkXxx(b *testing.B)

are considered benchmarks, and are executed by the “go test” command when >its -bench flag is provided. Benchmarks are run sequentially.
```
<!-- more -->
符合格式 （见上）的函数被认为是一个性能测试程序， 当带着 -bench=“.” （ 参数必须有！）来执行*go test命令的时候性能测试程序就会被顺序执行。

## 简单的实例代码

`lib.go`

```go
package lib

func Add(x int, y int) (z int) {
    z = x + y
    return
}


type ForTest struct {
    num int ;
}

func (this * ForTest) Loops() {
    for i:=0 ; i  < 10000 ; i++ {
        this.num++
    }
}
```

`lib_test.go`

```go
package lib
import "testing"

type AddArray struct {
    result  int;
    add_1   int;
    add_2   int;
}



func BenchmarkLoops(b *testing.B) {
    var test ForTest
    ptr := &test
    // 必须循环 b.N 次 。 这个数字 b.N 会在运行中调整，以便最终达到合适的时间消耗。方便计算出合理的数据。 （ 免得数据全部是 0 ） 
    for i:=0 ; i<b.N ; i++ {
        ptr.Loops()
    }
}
// 测试并发效率
func BenchmarkLoopsParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        var test ForTest
        ptr := &test
        for pb.Next() {
            ptr.Loops()
        }
    }
}

func TestAdd(t * testing.T) {
    var test_data = [3] AddArray {
        { 2, 1, 1},
        { 5, 2, 3},
        { 4, 2, 2},
    }
    for _ , v := range test_data {
        if v.result != Add(v.add_1, v.add_2) {
            t.Errorf("Add( %d , %d ) != %d \n", v.add_1 , v.add_2 , v.result);
        }
    }
}
```

执行 go test -bench=”.” 后 结果 ：

```
// 表示测试全部通过
>PASS                       
// Benchmark 名字 - CPU            循环次数             平均每次执行时间 
BenchmarkLoops-2                  100000             20628 ns/op     
BenchmarkLoopsParallel-2          100000             10412 ns/op   
//      哪个目录下执行go test         累计耗时
ok      swap/lib                   2.279s
```

## Go testing 库 testing.T 和 testing.B 简介

### testing.T
- 判定失败接口
	- Fail 失败继续
	- FailNow 失败终止
- 打印信息接口 
	- Log 数据流（cout　类似）
	- Logf format (printf 类似）
- SkipNow 跳过当前测试
- Skiped 检测是否跳过

### 综合接口产生

- Error / Errorf 报告出错继续 [ Log / Logf + Fail ]
- Fatel / Fatelf 报告出错终止 [ Log / Logf + FailNow ]
- Skip / Skipf 报告并跳过 [ Log / Logf + SkipNow ]

### testing.B

- testing.B 拥有testing.T 的全部接口
- SetBytes(i uint64) 统计内存消耗， 如果你需要的话
- SetParallelism(p int) 制定并行数目
- StartTimer / StopTimer / ResertTimer 操作计时器

### testing.PB

- Next()接口。判断是否继续循环

本文引自[这里](http://studygolang.com/articles/5494)

 
 