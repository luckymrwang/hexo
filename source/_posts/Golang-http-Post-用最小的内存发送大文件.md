title: Golang http.Post 用最小的内存发送大文件
date: 2019-11-16 21:01:36
tags: [Go]
---

在网络编程中，经常用http.Post 发送文件到远程服务器，可以通过自己构造multipart/form-data; boundary来实现。

一般是这么做：

```go
buf := new(bytes.Buffer)
writer := multipart.NewWriter(buf)
defer writer.Close()
part, err := writer.CreateFormFile("myFile", "foo.txt")
if err != nil {
    return err
}
file, err := os.Open(name)
if err != nil {
    return err
}
defer file.Close()
if _, err = io.Copy(part, file); err != nil {
    return err
}
http.Post(url, writer.FormDataContentType(), buf)
```

<!-- more -->

上面的代码是把整个文件读到buf 里，当文件很大时，就会占用很多内存。在golang 里可以使用io.Pipe 来优化内存占用：

```go
r, w := io.Pipe()
m := multipart.NewWriter(w)
go func() {
    defer w.Close()
    defer m.Close()
    part, err := m.CreateFormFile("myFile", "foo.txt")
    if err != nil {
        return
    }
    file, err := os.Open(name)
    if err != nil {
        return
    }
    defer file.Close()
    if _, err = io.Copy(part, file); err != nil {
        return
    }
}()
http.Post(url, m.FormDataContentType(), r)
```