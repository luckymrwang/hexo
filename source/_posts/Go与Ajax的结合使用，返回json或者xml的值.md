title: Go与Ajax的结合使用，返回json或者xml的值
date: 2016-07-31 15:44:48
tags: [Go]
toc: true
---
很多时候利用Go开发API的服务，会返回给调用端json或者xml格式的数据，这个在Go里面很容易做到。当然还有一些框架也可以轻松做到，比如[beego](http://beego.me/)，这里我们看一下标准库的代码如何做到。

<!-- more -->
```go
package main

import (
    "encoding/json"
    "encoding/xml"
    "fmt"
    "net/http"
)

type Dog struct {
    Name   string `json:"name" xml:"Name"`
    Age    int    `json:"age" xml:"Age"`
    Gender string `json:"gender" xml:"Gender"`
}

func main() {
    dog := Dog{
        Name:   "jemygraw",
        Age:    26,
        Gender: "male",
    }
    addr := "localhost:9090"
    http.HandleFunc("/xml", func(resp http.ResponseWriter, req *http.Request) {
        resp.Header().Set("Content-Type", "application/xml;charset=utf-8")
        data, _ := xml.Marshal(&dog)
        resp.Write(data)
    })

    http.HandleFunc("/json", func(resp http.ResponseWriter, req *http.Request) {
        resp.Header().Set("Content-Type", "application/json;charset=utf-8")
        data, _ := json.Marshal(&dog)
        resp.Write(data)
    })
    err := http.ListenAndServe(addr, nil)
    if err != nil {
        fmt.Println(err)
    }
}
```

主要步骤是设置回复的头部并写入相应格式的数据。这个在`beego`里面更加简单。

首先定义一个`MainController`:

```go
package controllers

import (
    "github.com/astaxie/beego"
)

type MainController struct {
    beego.Controller
}

type Dog struct {
    Name   string `json:"name" xml:"Name"`
    Age    int    `json:"age" xml:"Age"`
    Gender string `json:"gender" xml:"Gender"`
}

func (this *MainController) GetJson() {
    dog := Dog{
        Name:   "jemygraw",
        Age:    26,
        Gender: "male",
    }

    this.Data["json"] = &dog
    this.ServeJson()
}

func (this *MainController) GetXml() {
    dog := Dog{
        Name:   "jemygraw",
        Age:    26,
        Gender: "male",
    }

    this.Data["xml"] = &dog
    this.ServeXml()
}
```

然后定义`Router`:

```go
beego.Router("/json", &controllers.MainController{}, "get:GetJson")
beego.Router("/xml", &controllers.MainController{}, "get:GetXml")
```

这样就可以了。

相比之下，beego的做法更加符合Web开发的业务架构，所以可以考虑选用。

本文引自[这里](http://golanghome.com/post/870)
