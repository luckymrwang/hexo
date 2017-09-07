title: Golang中TCP Socket粘包问题和处理
date: 2016-11-15 12:21:58
tags: [Go]
toc: true
---

在用golang开发人工客服系统的时候碰到了粘包问题，那么什么是粘包呢？例如我们和客户端约定数据交互格式是一个json格式的字符串：

		{"Id":1,"Name":"golang","Message":"message"}

当客户端发送数据给服务端的时候，如果服务端没有及时接收，客户端又发送了一条数据上来，这时候服务端才进行接收的话就会收到两个连续的字符串，形如：

		{"Id":1,"Name":"golang","Message":"message"}{"Id":1,"Name":"golang","Message":"message"}
		
如果接收缓冲区满了的话，那么也有可能接收到半截的json字符串，酱紫的话还怎么用json解码呢？真是头疼。以下用golang模拟了下这个粘包的产生。

<!-- more -->
### 粘包示例

`server.go`

```go
//粘包问题演示服务端

package main

import (
	"fmt"

	"net"

	"os"
)

func main() {

	netListen, err := net.Listen("tcp", ":9988")

	CheckError(err)

	defer netListen.Close()

	Log("Waiting for clients")

	for {

		conn, err := netListen.Accept()

		if err != nil {

			continue

		}

		Log(conn.RemoteAddr().String(), " tcp connect success")

		go handleConnection(conn)

	}

}

func handleConnection(conn net.Conn) {

	buffer := make([]byte, 1024)

	for {

		n, err := conn.Read(buffer)

		if err != nil {

			Log(conn.RemoteAddr().String(), " connection error: ", err)

			return

		}

		Log(conn.RemoteAddr().String(), "receive data length:", n)

		Log(conn.RemoteAddr().String(), "receive data:", buffer[:n])

		Log(conn.RemoteAddr().String(), "receive data string:", string(buffer[:n]))

	}

}

func Log(v ...interface{}) {

	fmt.Println(v...)

}

func CheckError(err error) {

	if err != nil {

		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())

		os.Exit(1)

	}

}
```

`client.go`

```go
//粘包问题演示客户端

package main

import (
	"fmt"

	"net"

	"os"

	"time"
)

func sender(conn net.Conn) {

	for i := 0; i < 100; i++ {

		words := "{\"Id\":1,\"Name\":\"golang\",\"Message\":\"message\"}"

		conn.Write([]byte(words))

	}

}

func main() {

	server := "127.0.0.1:9988"

	tcpAddr, err := net.ResolveTCPAddr("tcp4", server)

	if err != nil {

		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())

		os.Exit(1)

	}

	conn, err := net.DialTCP("tcp", nil, tcpAddr)

	if err != nil {

		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())

		os.Exit(1)

	}

	defer conn.Close()

	fmt.Println("connect success")

	go sender(conn)

	for {

		time.Sleep(1 * 1e9)

	}

}
```
![粘包](/images/golang_stick.png)
可以看到json格式的字符串都粘到一起

### 粘包产生原因

关于粘包的产生原因网上有很多相关的说明，主要原因就是tcp数据传递模式是流模式，在保持长连接的时候可以进行多次的收和发。如果要深入了解可以看看tcp协议方面的内容。这里推荐下鸟哥的私房菜，讲的非常通俗易懂。

### 粘包解决办法

- 客户端发送一次就断开连接，需要发送数据的时候再次连接，典型如http。下面用golang演示一下这个过程，确实不会出现粘包问题

```go
//客户端代码，演示了发送一次数据就断开连接的

package main

import (
	"fmt"

	"net"

	"os"

	"time"
)

func main() {

	server := "127.0.0.1:9988"

	for i := 0; i < 100; i++ {

		tcpAddr, err := net.ResolveTCPAddr("tcp4", server)

		if err != nil {

			fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())

			os.Exit(1)

		}

		conn, err := net.DialTCP("tcp", nil, tcpAddr)

		if err != nil {

			fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())

			os.Exit(1)

		}

		words := "{\"Id\":1,\"Name\":\"golang\",\"Message\":\"message\"}"

		conn.Write([]byte(words))

		conn.Close()

	}

	for {

		time.Sleep(1 * 1e9)

	}

}
```

服务端代码参考上面演示粘包产生过程的服务端代码

- 包头+数据的格式，根据包头信息读取到需要分析的数据。形式如下图：

![粘包](/images/golang_stick2.jpg)

golang粘包问题包头定义

从数据流中读取数据的时候，只要根据包头和数据长度就能取到需要的数据。这个其实就是平时说的协议（protocol），只是这个数据传输协议非常简单，不像tcp、ip等协议有较多的定义。在实际的过程中通常会定义协议类或者协议文件来封装封包和解包的过程。下面代码演示了封包和解包的过程：

`protocal.go`

```go
//通讯协议处理，主要处理封包和解包的过程
package protocol

import (
	"bytes"
	"encoding/binary"
)

const (
	ConstHeader         = "www.01happy.com"
	ConstHeaderLength   = 15
	ConstSaveDataLength = 4
)

//封包
func Packet(message []byte) []byte {
	return append(append([]byte(ConstHeader), IntToBytes(len(message))...), message...)
}

//解包
func Unpack(buffer []byte, readerChannel chan []byte) []byte {
	length := len(buffer)

	var i int
	for i = 0; i < length; i = i + 1 {
		if length < i+ConstHeaderLength+ConstSaveDataLength {
			break
		}
		if string(buffer[i:i+ConstHeaderLength]) == ConstHeader {
			messageLength := BytesToInt(buffer[i+ConstHeaderLength : i+ConstHeaderLength+ConstSaveDataLength])
			if length < i+ConstHeaderLength+ConstSaveDataLength+messageLength {
				break
			}
			data := buffer[i+ConstHeaderLength+ConstSaveDataLength : i+ConstHeaderLength+ConstSaveDataLength+messageLength]
			readerChannel <- data

			i += ConstHeaderLength + ConstSaveDataLength + messageLength - 1
		}
	}

	if i == length {
		return make([]byte, 0)
	}
	return buffer[i:]
}

//整形转换成字节
func IntToBytes(n int) []byte {
	x := int32(n)

	bytesBuffer := bytes.NewBuffer([]byte{})
	binary.Write(bytesBuffer, binary.BigEndian, x)
	return bytesBuffer.Bytes()
}

//字节转换成整形
func BytesToInt(b []byte) int {
	bytesBuffer := bytes.NewBuffer(b)

	var x int32
	binary.Read(bytesBuffer, binary.BigEndian, &x)

	return int(x)
}
```

tips：解包的过程中要注意数组越界的问题；另外包头要注意唯一性

`server.go`

```go
//服务端解包过程
package main

import (
	"./protocol"
	"fmt"
	"net"
	"os"
)

func main() {
	netListen, err := net.Listen("tcp", ":9988")
	CheckError(err)

	defer netListen.Close()

	Log("Waiting for clients")
	for {
		conn, err := netListen.Accept()
		if err != nil {
			continue
		}

		Log(conn.RemoteAddr().String(), " tcp connect success")
		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
	//声明一个临时缓冲区，用来存储被截断的数据
	tmpBuffer := make([]byte, 0)

	//声明一个管道用于接收解包的数据
	readerChannel := make(chan []byte, 16)
	go reader(readerChannel)

	buffer := make([]byte, 1024)
	for {
		n, err := conn.Read(buffer)
		if err != nil {
			Log(conn.RemoteAddr().String(), " connection error: ", err)
			return
		}

		tmpBuffer = protocol.Unpack(append(tmpBuffer, buffer[:n]...), readerChannel)
	}
}

func reader(readerChannel chan []byte) {
	for {
		select {
		case data := <-readerChannel:
			Log(string(data))
		}
	}
}

func Log(v ...interface{}) {
	fmt.Println(v...)
}

func CheckError(err error) {
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}
}
```

`client.go`

```go
//客户端发送封包
package main

import (
	"./protocol"
	"fmt"
	"net"
	"os"
	"time"
)

func sender(conn net.Conn) {
	for i := 0; i < 1000; i++ {
		words := "{\"Id\":1,\"Name\":\"golang\",\"Message\":\"message\"}"
		conn.Write(protocol.Packet([]byte(words)))
	}
	fmt.Println("send over")
}

func main() {
	server := "127.0.0.1:9988"
	tcpAddr, err := net.ResolveTCPAddr("tcp4", server)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}

	conn, err := net.DialTCP("tcp", nil, tcpAddr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}

	defer conn.Close()
	fmt.Println("connect success")
	go sender(conn)
	for {
		time.Sleep(1 * 1e9)
	}
}
```

运行这个程序可以看到服务端很好的获取到期望的json格式数据 

[完整代码展示](https://github.com/luckymrwang/hello/tree/master/tests/example/golang%E7%B2%98%E5%8C%85%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E7%A4%BA%E4%BE%8B)

### 最后

上面演示的两种方法适用于不同的场景。第一种方法比较适合被动型的场景，例如打开网页，用户有请求才处理交互。第二种方法适合于主动推送的类型，例如即时聊天系统，因为要即时给用户推送消息，保持长连接是不可避免的，这时候就要用这种方法。

本文引自[这里](http://studygolang.com/articles/8917)

