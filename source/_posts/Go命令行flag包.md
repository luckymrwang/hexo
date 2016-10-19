title: Go命令行flag包
date: 2016-09-27 18:07:02
tags: [Go笔记]
---

go语言提供的flag包可以解析命令行的参数，代码：

```go
package main

import (
	"flag"

	"fmt"
)

func main() {

	//第一个参数，为参数名称，第二个参数为默认值，第三个参数是说明

	username := flag.String("name", "", "Input your username")

	flag.Parse()

	fmt.Println("Hello, ", *username)

}
```

编译：

	go build flag.go

运行：

	./flag -name=world

输出：

	Hello, world

如果不输入name参数：

	./flag

则输出：

	Hello,
