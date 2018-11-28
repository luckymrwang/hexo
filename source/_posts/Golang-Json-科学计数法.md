title: Golang Json 科学计数法
date: 2018-11-28 14:29:57
tags: [Golang]
---

使用Go解析一个从其他接口返回的JSON字符串，有时会遇到数字以科学计数法的形式展现，比如

源：

```json
{"message": "success", "data": 6566651968}
```

处理后：

```json
{"message": "success", "data": 6.566651968e+09}
```
<!-- more -->
Go程序大体如下：

```go
func xxxx() {
    // ......
    v := make(map[string]interface{})

    // body是后端的http返回结果
    err = json.Unmarshal(body, &v)
    if err != nil {
        // 错误处理
    }
    
    ...
}
```

例如Python遇到普通数值解析出来为int, 碰到科学技术法解析出来则为float:

```python
>>> import json

# 普通数值

>>> d = json.loads('{"retval": 0, "message": "success", "data": 5678000}')
>>> d['data']
5678000
>>> type(d['data'])
<type 'int'>


# 科学计数法

>>> d = json.loads('{"retval": 0, "message": "success", "data": 5.678E7}')
>>> d['data']
56780000.0
>>> type(d['data'])
<type 'float'>
```

为什么会这么处理? Go官方文档的说明:

>To unmarshal JSON into an interface value, Unmarshal stores one of these in the interface value:
>
>float64, for JSON numbers

解决方案是使用[Decoder.UseNumber](https://golang.org/pkg/encoding/json/#Decoder.UseNumber)方法, 可以简单看下这个[回答](https://stackoverflow.com/questions/22343083/json-marshaling-with-long-numbers-in-golang-gives-floating-point-number)

修改后Go代码:

```go
func xxxx() {
    // ......
    v := make(map[string]interface{})

    // body是后端的http返回结果
    d := json.NewDecoder(bytes.NewReader(body))
    d.UseNumber()
    err = d.Decode(&v)
    if err != nil {
        // 错误处理
    }

    ...
}
```