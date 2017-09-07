title: Go语言的defer
date: 2017-06-16 16:39:16
tags: [Go]
---
### eg1
```go
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```

### eg2
```go
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```
<!-- more -->
### eg3
```go
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}
```

明确一点：`defer是在return之前执行的`

`defer的实现方式`大致就是在defer出现的地方，插入指令CALL runtime.deferproc，然后在函数返回之前的地方，插入指令CALL runtime.deferreturn。再就是明确go返回值的方式跟C是不一样的，为了支持多值返回，go是用栈返回值的，而C是用寄存器。

重要的一点就是：`return xxx这一条语句并不是一条原子指令`

整个return过程，没有defer之前，先在栈中写一个值，这个值会被当作返回值，然后再调用RET指令返回。return xxx语句汇编后是先给返回值赋值，再做一个空的return，( 赋值指令 ＋ RET指令)。defer的执行是被插入到return指令之前的，有了defer之后，就变成了(赋值指令 + CALL defer指令 + RET指令)。而在CALL defer函数中，有可能将最终的返回值改写了...也有可能没改写。总之，如果改写了，那么看上去就像defer是在return xxx之后执行的~

可以利用简单的转换规则，将return语句分开成两句写，return xxx会被改写成:
```
返回值 = xxx
调用defer函数
空的return
```

### eg1 改写后
```go
func f() (result int) {
     result = 0  //return语句不是一条原子调用，return xxx其实是赋值＋RET指令
     func() { //defer被插入到return之前执行，也就是赋返回值和RET指令之间
         result++
     }()
     return
}
```
返回值为1

### eg2 改写后
```go
func f() (r int) {
     t := 5
     r = t //赋值指令
     func() {        //defer被插入到赋值与返回之间执行，这个例子中返回值r没被修改过
         t = t + 5
     }
     return        //空的return指令
}
```
返回值为5

### eg3 改写后
```go
func f() (r int) {
     r = 1  //给返回值赋值
     func(r int) {        //这里改的r是传值传进去的r，不会改变要返回的那个r值
          r = r + 5
     }(r)
     return        //空的return
}
```
返回值为1

本文引自[这里](http://www.zenlife.tk/golang-defer.md)
