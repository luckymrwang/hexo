title: Go Zero Copy Transfer
date: 2022-10-19 10:37:02
tags: [Go,Linux]
---
平时写 `http server` 会接触到传输文件的问题。最简单的版本大概是这样子的：

```go
for {
	n, _ := file.Read(buf)
	if n > 0 {
		w.Write(buf[:n])
	} else {
		break
	}
}
```
<!-- more -->

对于一般的程序来说是没有问题的，但是如果需要频繁地传输大文件，那么这种办法效率不够高。因为这里不断地调用两个系统调用，`read()` 和 `write()`。对操作系统稍微熟悉的都会知道系统调用是在**内核态**执行而用户程序是跑在**用户态**。

读取文件就会先从磁盘文件读到内核缓冲区然后再拷贝到用户空间缓冲区，写操作也要先写到内核缓冲区然后再发送。

![image](/images/zerocopy1.png)

可以清楚看到这里一共触发了 4 次用户态和内核态的上下文切换，分别是 `read()/write()` 调用和返回时的切换，2 次 DMA 拷贝，2 次 CPU 拷贝，加起来一共 4 次拷贝操作。

通过引入 DMA，我们已经把 Linux 的 I/O 过程中的 CPU 拷贝次数从 4 次减少到了 2 次，但是 CPU 拷贝依然是代价很大的操作，对系统性能的影响还是很大，特别是那些频繁 I/O 的场景，更是会因为 CPU 拷贝而损失掉很多性能，我们需要进一步优化，降低、甚至是完全避免 CPU 拷贝。

系统调用 `sendfile()` 就是用来解决这个底性能问题的。文件可以直接送到 `socket` 上，不需要经过用户空间。

![image](/images/zorecopy2.png)

使用 `sendfile()` 完成一次数据读写的流程如下：

1、用户进程调用 sendfile() 从用户态陷入内核态；

2、DMA 控制器将数据从硬盘拷贝到内核缓冲区；

3、CPU 将内核缓冲区中的数据拷贝到套接字缓冲区；

4、DMA 控制器将数据从套接字缓冲区拷贝到网卡完成数据传输；

5、sendfile() 返回，上下文从内核态切换回用户态。

### sendfile()

```c
# include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

`in_fd` 是代表输入文件的文件描述符，`out_fd` 是代表输出文件的文件描述符。`out_id` 必须为 socket (linux 2.6.33 开始可以是任何文件)。 `in_fd` 指向的文件必须为可以进行 mmap() 操作的，通常为普通文件，不能是 Socket 类型。

### splice()

Linux 在 2.6.17 版本引入了一个新的系统调用 `splice()`，它在功能上和 `sendfile()` 非常相似，但是能够实现在任意类型的两个文件描述符时之间传输数据；而在底层实现上，`splice()` 又比 `sendfile()` 少了一次 CPU 拷贝，也就是等同于 `sendfile() + DMA Scatter/Gather`，完全去除了数据传输过程中的 CPU 拷贝。

```c
#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t *off_in, int fd_out,
               loff_t *off_out, size_t len, unsigned int flags);
```

![image](/images/zorecopy3.png)

使用 `splice()` 完成一次磁盘文件到网卡的读写过程如下：

1、用户进程调用 pipe()，从用户态陷入内核态，创建匿名单向管道，pipe() 返回，上下文从内核态切换回用户态；

2、用户进程调用 splice()，从用户态陷入内核态；

3、DMA 控制器将数据从硬盘拷贝到内核缓冲区，从管道的写入端"拷贝"进管道，splice() 返回，上下文从内核态回到用户态；

4、用户进程再次调用 splice()，从用户态陷入内核态；

5、内核把数据从管道的读取端"拷贝"到套接字缓冲区，DMA 控制器将数据从套接字缓冲区拷贝到网卡；

6、splice() 返回，上下文从内核态切换回用户态。

### io.Copy()

当我们需要响应请求并返回文件时，我们可以使用 `io.Copy()`，因为底层使用了上面提及的 `splice()` 和 `sendfile()` 系统调用。

```go
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}

func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if buf != nil && len(buf) == 0 {
		panic("empty buffer in io.CopyBuffer")
	}
	return copyBuffer(dst, src, buf)
}

func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
......
```

上面的源码中可以看到，两个 `type assertion`。 现在关注一下第二个 `dst.(ReaderFrom)`。

因为我们传参第一个是 `http.ResponseWriter`，所以实现这个接口的 `struct` 是下面这个。

```go
// A response represents the server side of an HTTP response.
type response struct {
	conn             *conn
	req              *Request // request for this response
	reqBody          io.ReadCloser
	cancelCtx        context.CancelFunc // when ServeHTTP exits
	wroteHeader      bool               // reply header has been (logically) written
	wroteContinue    bool               // 100 Continue response was written
	wants10KeepAlive bool               // HTTP/1.0 w/ Connection "keep-alive"
	wantsClose       bool               // HTTP request has Connection "close"

	w  *bufio.Writer // buffers output in chunks to chunkWriter
	cw chunkWriter

......
```

找到具体实现 `ReadFrom` 的地方

```go
// ReadFrom is here to optimize copying from an *os.File regular file
// to a *net.TCPConn with sendfile.
func (w *response) ReadFrom(src io.Reader) (n int64, err error) {
	// Our underlying w.conn.rwc is usually a *TCPConn (with its
	// own ReadFrom method). If not, or if our src isn't a regular
	// file, just fall back to the normal copy method.
	rf, ok := w.conn.rwc.(io.ReaderFrom)
	regFile, err := srcIsRegularFile(src)
	if err != nil {
		return 0, err
	}
	if !ok || !regFile {
		bufp := copyBufPool.Get().(*[]byte)
		defer copyBufPool.Put(bufp)
		return io.CopyBuffer(writerOnly{w}, src, *bufp)
	}

	// sendfile path:

	if !w.wroteHeader {
		w.WriteHeader(StatusOK)
	}

	if w.needsSniff() {
		n0, err := io.Copy(writerOnly{w}, io.LimitReader(src, sniffLen))
		n += n0
		if err != nil {
			return n, err
		}
	}

	w.w.Flush()  // get rid of any previous writes
	w.cw.flush() // make sure Header is written; flush data to rwc

	// Now that cw has been flushed, its chunking field is guaranteed initialized.
	if !w.cw.chunking && w.bodyAllowed() {
		n0, err := rf.ReadFrom(src)
		n += n0
		w.written += n0
		return n, err
	}

	n0, err := io.Copy(writerOnly{w}, src)
	n += n0
	return n, err
}
```

函数开头有一段注释已经说明调用的底层是 `*TCPConn` 的 `ReadFrom`。

```go
// ReadFrom implements the io.ReaderFrom ReadFrom method.
func (c *TCPConn) ReadFrom(r io.Reader) (int64, error) {
	if !c.ok() {
		return 0, syscall.EINVAL
	}
	n, err := c.readFrom(r)
	if err != nil && err != io.EOF {
		err = &OpError{Op: "readfrom", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
	}
	return n, err
}
```

终于出现了 `splice()` 和 `sendFile()`

```go
func (c *TCPConn) readFrom(r io.Reader) (int64, error) {
	if n, err, handled := splice(c.fd, r); handled {
		return n, err
	}
	if n, err, handled := sendFile(c.fd, r); handled {
		return n, err
	}
	return genericReadFrom(c, r)
}
```

再底层的就是 `go` 对系统调用 `splice()` 和 `sendFile()` 的封装了。

### 测试 `sendfile`
以 [caddy](https://github.com/caddyserver/caddy) 项目为例，测试 `io.Copy()` 是否支持 `sendfile` 系统调用。

1. 构造两个文件，任何超过 512 字节的文件都应该在 http1 中使用 `sendfile`。

用 `dd` 命令向文件中写指定大小的数据：

```sh
# 512B
dd if=/dev/zero bs=512 count=1 > nosendfile.html
# 513B
dd if=/dev/zero bs=513 count=1 > sendfile.html
# 查看
wc -c *.html
 512 nosendfile.html
 513 sendfile.html
```

2. 检测系统调用

```sh
strace -f -e sendfile ./caddy file-server --root . --listen 127.0.0.1:8080 --access-log

strace: Process 137302 attached
[...]
2022/09/06 21:56:45.335 INFO    Caddy serving static files on 127.0.0.1:8080
2022/09/06 21:56:50.505 INFO    http.log.access handled request {"request": {"proto": "HTTP/1.1", "method": "GET", "uri": "/nosendfile.html", "size": 512, "status": 200}}
[pid 137301] sendfile(7, 8, NULL, 1)    = 1
2022/09/06 21:57:00.281 INFO    http.log.access handled request {"request": {"proto": "HTTP/1.1", "method": "GET", "uri": "/sendfile.html", "size": 513, "status": 200}}
```

```sh
curl 127.0.0.1:8080/nosendfile.html
curl 127.0.0.1:8080/sendfile.html
```

本文引用

- [zero copy transfer](https://delveshal.github.io/2018/08/31/zero-copy-transfer/)
- [Linux I/O 原理和 Zore-copy 技术全面揭秘](https://strikefreedom.top/linux-io-and-zero-copy)
- [Go 语言中的零拷贝优化](https://strikefreedom.top/archives/pipe-pool-for-splice-in-go)