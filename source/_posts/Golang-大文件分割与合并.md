title: Golang 大文件分割与合并
date: 2019-11-16 21:29:17
tags: [Go]
---

大文件分割与合并例子

<!--more-->

```go
package main

import (
	"fmt"
	"io/ioutil"
	"math"
	"os"
	"strconv"
)

const chunkSize = 1 << (10 * 2)

func main() {
	fileInfo, err := os.Stat("cbd.mp4")
	if err != nil {
		panic(err)
	}

	num := math.Ceil(float64(fileInfo.Size()) / chunkSize)

	fi, err := os.OpenFile("cbd.mp4", os.O_RDONLY, os.ModePerm)
	if err != nil {
		fmt.Println(err)
		return
	}
	b := make([]byte, chunkSize)
	var i int64 = 1
	for ; i <= int64(num); i++ {
		fi.Seek((i-1)*chunkSize, 0)
		if len(b) > int(fileInfo.Size()-(i-1)*chunkSize) {
			b = make([]byte, fileInfo.Size()-(i-1)*chunkSize)
		}
		fi.Read(b)

		f, err := os.OpenFile("./"+strconv.Itoa(int(i))+".db", os.O_CREATE|os.O_WRONLY, os.ModePerm)
		if err != nil {
			panic(err)
		}
		f.Write(b)
		f.Close()
	}
	fi.Close()

	fii, err := os.OpenFile("all.mp4", os.O_CREATE|os.O_WRONLY|os.O_APPEND, os.ModePerm)
	if err != nil {
		panic(err)
		return
	}
	for i := 1; i <= int(num); i++ {
		f, err := os.OpenFile("./"+strconv.Itoa(int(i))+".db", os.O_RDONLY, os.ModePerm)
		if err != nil {
			fmt.Println(err)
			return
		}
		b, err := ioutil.ReadAll(f)
		if err != nil {
			fmt.Println(err)
			return
		}
		fii.Write(b)
		f.Close()
	}
}
```