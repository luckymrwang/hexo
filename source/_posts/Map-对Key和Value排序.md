title: Map 对Key和Value排序
date: 2017-03-21 11:32:35
tags: [Go]
toc: true
---
### Map 根据键名排序

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	// To create a map as input
	m := make(map[int]string)
	m[1] = "a"
	m[2] = "c"
	m[0] = "b"

	// To store the keys in slice in sorted order
	var keys []int
	for k := range m {
		keys = append(keys, k)
	}
	sort.Ints(keys)

	// To perform the opertion you want
	for _, k := range keys {
		fmt.Println("Key:", k, "Value:", m[k])
	}
}
```
<!-- more -->

### Map 根据键值排序

```go
package tools

import (
	"sort"
)

type Map struct {
	Key   string
	Value int
}

type MapSlice []Map

func (m MapSlice) Len() int           { return len(m) }
func (m MapSlice) Less(i, j int) bool { return m[i].Value < m[j].Value }
func (m MapSlice) Swap(i, j int)      { m[i], m[j] = m[j], m[i] }

func Sort(m map[string]int) MapSlice {
	ms := make(MapSlice, 0)
	for k, v := range m {
		ms = append(ms, Map{k, v})
	}
	sort.Sort(ms)

	return ms
}

func SortReverse(m map[string]int) MapSlice {
	ms := make(MapSlice, 0)
	for k, v := range m {
		ms = append(ms, Map{k, v})
	}
	sort.Sort(sort.Reverse(ms))

	return ms
}
```
