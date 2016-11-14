title: 用Golang实现 服务器/客户端
date: 2016-11-13 20:59:16
tags: [Go]
toc: true
---

### 服务端的实现思路及步骤

1. 创建一个套接字对象, 指定其IP以及端口
2. 开始监听套接字指定的端口
3. 如有新的客户端连接请求, 则建立一个goroutine, 在goroutine中, 读取客户端消息, 并转发回去, 直到客户端断开连接
4. 主进程继续监听端口

<!-- more -->
服务器程序 `server.go` 代码如下：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"time"
)

func main() {
	var tcpAddr *net.TCPAddr

	tcpAddr, _ = net.ResolveTCPAddr("tcp", "127.0.0.1:9999")

	tcpListener, _ := net.ListenTCP("tcp", tcpAddr)

	defer tcpListener.Close()

	for {
		tcpConn, err := tcpListener.AcceptTCP()
		if err != nil {
			continue
		}

		fmt.Println("A client connected : " + tcpConn.RemoteAddr().String())
		go tcpPipe(tcpConn)
	}

}

func tcpPipe(conn *net.TCPConn) {
	ipStr := conn.RemoteAddr().String()
	defer func() {
		fmt.Println("disconnected :" + ipStr)
		conn.Close()
	}()
	reader := bufio.NewReader(conn)

	for {
		message, err := reader.ReadString('\n')
		if err != nil {
			return
		}

		fmt.Println(string(message))
		msg := time.Now().String() + "\n"
		b := []byte(msg)
		conn.Write(b)
	}
}
```

### 客户端的代码实现步骤

1. 创建一个套接字对象, ip与端口指定到上面我们实现的服务器的ip与端口上
2. 使用创建好的套接字对象连接服务器.
3. 连接成功后, 开启一个goroutine, 在这个goroutine内, 定时的向服务器发送消息, 并接受服务器的返回消息, 直到错误发生或断开连接

客户端程序 `client.go` 代码如下：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"time"
)

var quitSemaphore chan bool

func main() {
	var tcpAddr *net.TCPAddr
	tcpAddr, _ = net.ResolveTCPAddr("tcp", "127.0.0.1:9999")

	conn, _ := net.DialTCP("tcp", nil, tcpAddr)
	defer conn.Close()
	fmt.Println("connected!")

	go onMessageRecived(conn)

	b := []byte("time\n")
	conn.Write(b)

	<-quitSemaphore
}

func onMessageRecived(conn *net.TCPConn) {
	reader := bufio.NewReader(conn)
	for {
		msg, err := reader.ReadString('\n')
		fmt.Println(msg)
		if err != nil {
			quitSemaphore <- true
			break
		}
		time.Sleep(time.Second)
		b := []byte(msg)
		conn.Write(b)
	}
}
```

最后编译并分别运行 server 和 client

类似的输出：

```
connected! 2015-03-19 23:42:08.4875559 +0800 CST

2015-03-19 23:42:09.4896132 +0800 CST

2015-03-19 23:42:10.4906704 +0800 CST

2015-03-19 23:42:11.4917277 +0800 CST

2015-03-19 23:42:12.4927849 +0800 CST

2015-03-19 23:42:13.4938422 +0800 CST

2015-03-19 23:42:14.4948995 +0800 CST

2015-03-19 23:42:15.4959567 +0800 CST

2015-03-19 23:42:16.497014 +0800 CST

2015-03-19 23:42:17.4980712 +0800 CST

2015-03-19 23:42:18.4991285 +0800 CST

2015-03-19 23:42:19.5001857 +0800 CST
```




