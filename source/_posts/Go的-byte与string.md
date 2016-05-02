title: 'Go的[]byte与string'
date: 2016-03-06 12:56:50
tags: [Go]
toc: true
---
```go
package main

import (
    "fmt"
)

func main() {
    s1 := "abcd"
    b1 := []byte(s1)
    fmt.Println(b1) // [97 98 99 100]

    s2 := "中文"
    b2 := []byte(s2)
    fmt.Println(b2) // [228 184 173 230 150 135], unicode，每个中文字符会由三个byte组成 

    r := []rune(s2)
    fmt.Println(r) // [20013 25991], 每个字一个数值
}
```
<!-- more -->
byte 对应 utf-8 英文占1个字节，中文占3个字节

rune 对应 unicode 英文字符占2个字节，中文字符占2个字节

在Go中字符串是不可变的,例如下面的代码编译时会报错:
	var s string = "hello" s[0] = 'c'
	但如果真的想要修改怎么办呢?下面的代码可以实现:

```gos := "hello"c := []byte(s) // 将字符串 s 转换为 []byte 类型 c[0] = 'c's2 := string(c) // 再转换回 string 类型 fmt.Printf("%s\n", s2)
```Go中可以使用+操作符来连接两个字符串:

```gos := "hello,"m := " world"a := s + m fmt.Printf("%s\n", a)
```
修改字符串也可写为:

```gos := "hello"s = "c" + s[1:] // 字符串虽不能更改,但可进行切片操作 fmt.Printf("%s\n", s)
```