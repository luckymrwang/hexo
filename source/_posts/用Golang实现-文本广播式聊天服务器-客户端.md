title: 用Golang实现 文本广播式聊天服务器/客户端
date: 2016-11-13 21:07:48
tags: [Go]
toc: true
---

根据 [用Golang实现 服务器/客户端](https://luckymrwang.github.io/2016/11/13/%E7%94%A8Golang%E5%AE%9E%E7%8E%B0-%E6%9C%8D%E5%8A%A1%E5%99%A8-%E5%AE%A2%E6%88%B7%E7%AB%AF/) 将其改造成一个文本信息聊天室

### 服务端的改动

1. 服务器为了实现聊天信息的群体广播, 需要记录所有连接到服务器的客户端信息, 所以, 我们需要添加一个集合来保存所有客户端的连接:
		
		var ConnMap map[string]*net.TCPConn
		
2. 接着, 每次当有新的客户端连接到服务器时, 需要把这个客户端连接行信息加入集合:

		ConnMap[tcpConn.RemoteAddr().String()] = tcpConn
		
3. 当服务器收到客户端的聊天信息时, 需要广播到所有客户端, 所以我们需要利用上面保存TCPConn的map来遍历所有TCPConn进行广播, 用以下方法实现:

<!-- more -->
```go
func boradcastMessage(message string) {
     b := []byte(message)
     for _, conn := range ConnMap {
     conn.Write(b)
     }
 }
```

### 客户端代码改动

客户端代码改动相对简单, 只是加入了用户自己输入聊天信息的功能, 在连接成功并且启动了消息接收的gorountine后, 加入以下代码:

```go
for {
    var msg string
    fmt.Scanln(&msg)
    if msg == "quit" {
        break
    }
    b := []byte(msg + "\n")
    conn.Write(b)
}
```

完整的服务端 `server.go` 代码如下:

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

// 用来记录所有的客户端连接
var ConnMap map[string]*net.TCPConn

func main() {
	var tcpAddr *net.TCPAddr
	ConnMap = make(map[string]*net.TCPConn)
	tcpAddr, _ = net.ResolveTCPAddr("tcp", "127.0.0.1:9999")

	tcpListener, _ := net.ListenTCP("tcp", tcpAddr)

	defer tcpListener.Close()

	for {
		tcpConn, err := tcpListener.AcceptTCP()
		if err != nil {
			continue
		}

		fmt.Println("A client connected : " + tcpConn.RemoteAddr().String())
		// 新连接加入map
		ConnMap[tcpConn.RemoteAddr().String()] = tcpConn
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
		fmt.Println(conn.RemoteAddr().String() + ":" + string(message))
		// 这里返回消息改为了广播
		boradcastMessage(conn.RemoteAddr().String() + ":" + string(message))
	}
}

func boradcastMessage(message string) {
	b := []byte(message)
	// 遍历所有客户端并发送消息
	for _, conn := range ConnMap {
		conn.Write(b)
	}
}
```

客户端 `client.go` 完整代码如下:

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	var tcpAddr *net.TCPAddr
	tcpAddr, _ = net.ResolveTCPAddr("tcp", "127.0.0.1:9999")

	conn, _ := net.DialTCP("tcp", nil, tcpAddr)
	defer conn.Close()
	fmt.Println("connected!")

	go onMessageRecived(conn)

	// 控制台聊天功能加入
	for {
		var msg string
		fmt.Scanln(&msg)
		if msg == "quit" {
			break
		}
		b := []byte(msg + "\n")
		conn.Write(b)
	}
}

func onMessageRecived(conn *net.TCPConn) {
	reader := bufio.NewReader(conn)
	for {
		msg, err := reader.ReadString('\n')
		fmt.Println(msg)
		if err != nil {
			panic(err)
			break
		}
	}
}
```

最后分别编译

先启动server端, 然后新开两个终端, 启动客户端, 在其中一个客户端里键入聊天信息后回车, 会发现另外一个客户端收到了刚刚发送的聊天信息

### 服务器的粘包处理

#### 什么是粘包

一个完成的消息可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这个就是TCP的拆包和封包问题

#### TCP粘包和拆包产生的原因

1. 应用程序写入数据的字节大小大于套接字发送缓冲区的大小
2. 进行MSS大小的TCP分段。MSS是最大报文段长度的缩写。MSS是TCP报文段中的数据字段的最大长度。数据字段加上TCP首部才等于整个的TCP报文段。所以MSS并不是TCP报文段的最大长度，而是：MSS=TCP报文段长度-TCP首部长度
3. 以太网的payload大于MTU进行IP分片。MTU指：一种通信协议的某一层上面所能通过的最大数据包大小。如果IP层有一个数据包要传，而且数据的长度比链路层的MTU大，那么IP层就会进行分片，把数据包分成若干片，让每一片都不超过MTU。注意，IP分片可以发生在原始发送端主机上，也可以发生在中间路由器上

#### TCP粘包和拆包的解决策略

1. 消息定长。例如100字节
2. 在包尾部增加回车或者空格符等特殊字符进行分割，典型的如FTP协议
3. 将消息分为消息头和消息尾
4. 其它复杂的协议，如RTMP协议等

#### 处理方式

- 发送方在每次发送消息时将消息长度写入一个int32作为包头一并发送出去, 我们称之为Encode
- 接受方则先读取一个int32的长度的消息长度信息, 再根据长度读取相应长的byte数据, 称之为Decode

在实验环境中的主文件夹内, 新建一个名为codec的文件夹在其之下新建一个文件codec.go, 将Encode和Decode方法写入其中, 这里给出Encode与Decode相应的代码:

`codec.go`
```go
package codec

import (
    "bufio"
    "bytes"
    "encoding/binary"
)

func Encode(message string) ([]byte, error) {
    // 读取消息的长度
    var length int32 = int32(len(message))
    var pkg *bytes.Buffer = new(bytes.Buffer)
    // 写入消息头
    err := binary.Write(pkg, binary.LittleEndian, length)
    if err != nil {
        return nil, err
    }
    // 写入消息实体
    err = binary.Write(pkg, binary.LittleEndian, []byte(message))
    if err != nil {
        return nil, err
    }

    return pkg.Bytes(), nil
}

func Decode(reader *bufio.Reader) (string, error) {
    // 读取消息的长度
    lengthByte, _ := reader.Peek(4)
    lengthBuff := bytes.NewBuffer(lengthByte)
    var length int32
    err := binary.Read(lengthBuff, binary.LittleEndian, &length)
    if err != nil {
        return "", err
    }
    if int32(reader.Buffered()) < length+4 {
        return "", err
    }

    // 读取消息真正的内容
    pack := make([]byte, int(4+length))
    _, err = reader.Read(pack)
    if err != nil {
        return "", err
    }
    return string(pack[4:]), nil
}
```

本文引自[这里](https://victoriest.gitbooks.io/golang-tcp-server/content/chapter4.html)