title: Golang 在 Mac、Linux、Windows 下交叉编译
date: 2018-08-02 10:02:21
tags: [Go]
---
#### Mac 下编译 Linux 和 Windows 64位可执行程序
```go
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

#### Linux 下编译 Mac 和 Windows 64位可执行程序
```go
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

#### Windows 下编译 Mac 和 Linux 64位可执行程序

```go
SET CGO_ENABLED=0
SET GOOS=darwin
SET GOARCH=amd64
go build main.go

SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build main.go
```
