title: protobuf import路径问题
date: 2019-01-02 11:39:18
tags: [Protobuf]
---

比如有这样的一个目录结构

```js
--protobuf
	--main.proto
	--main.pb.go
	--storyboard
		--storyboard.proto
```

怎样在`storyboard.proto`中引入`main.proto`？

<!-- more -->
`main.proto`中的代码

```js
syntax = "proto3";

package main;

option go_package = "main";

// Request 请求结构
message Request {
	map<string,string> header = 1;
	map<string,string> args = 2;
}

// Response 响应结构
message Response {
	int64 code = 1;
    string message = 2;
}
```


***
执行`proto`编译生成`main.pb.go`文件：

```js
protoc -I . ./main.proto --go_out=plugins=grpc:.
```
***

`storyboard.proto`中的代码

```js
syntax = "proto3";

import "main.proto";
package storyboard;

option go_package = "storyboard";

// 定义服务
service Storyboard {
	// 定义方法
	rpc Get(main.Request) returns (main.Response) {}
	rpc List(main.Request) returns (main.Response) {}
}
```

执行`proto`编译生成`storyboard.pb.go`文件：

```js
protoc --proto_path=. --proto_path=storyboard/ --go_out=plugins=grpc:storyboard storyboard.proto
```

参数解析

- --proto_path: 指定了在哪个目录中搜索import中导入的和要编译为.go的proto文件，可以定义多个
- --go_out: 指定了生成的go文件的目录
- storyboard.proto, 要编译的文件是





