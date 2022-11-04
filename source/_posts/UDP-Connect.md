title: UDP Connect
date: 2022-10-27 22:36:41
tags: [Linux,UDP]
---
在 UDP 上使用 `connect` 的情况：

1. 需要获取 ICMP 的错误信息
2. 如果需要向同一个IP地址多次 `sendto` ，用以减少不断的连接、断开，提高性能

<!-- more -->
注意：

UDP 的 `connect` 只记录对端的套接字结构 `(ip, port)`，并不像 TCP 进行3次握手，所以无法第一时间获取连接错误。这种称为有连接的 `UDP` 套接字也只能 发送/接受 `connect` 中的指定的 ip 和 port。

下面一个例子说明了 `sendto` 只是向内核缓冲区复制数据就返回了，并不会产生 `ICMP` 错误信息。

### 无 `connect`

```c
int main(int argc, char**argv)
{
    int sockfd  = socket(AF_INET,SOCK_DGRAM,0); //udp
    struct sockaddr_in sin;
    memset(&sin,0,sizeof(sin));
    sin.sin_port = htons(PORT);    //没有服务器
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = inet_addr("127.0.0.1"); 
    char send[100],recv[100];
    int n = 0;
 
    while(1){
        n = read(0,send,100);
        n =sendto(sockfd,send,n,0,(SA*)&sin,sizeof(sin)); //发送到缓冲区就返回, 没有icmp错误信息
        printf("sendto return : %d\n" , n);
        n = recvfrom(sockfd,recv,100,0,0,0); // 此时将一直阻塞在这里 等待接受数据
        printf("recvfrom return :%d\n",n);
        recv[n]  =0;
        printf("buf : %s\n", recv);
    }
 
    return 0;
}
```

通过调用 `connect` 后的代码, 把 `sendto` , `recvfrom` 换成了 `read` , `write` (不是一定要换,只是后2个函数参数少)

通过 `connect` 后的 UDP 套接字可以收到 `ICMP` 错误信息了, 在 `read` 返回后将产生错误, `write` 仅仅是复制数据到缓冲区;

```c
#include "util.h"
#include <netdb.h>
extern int h_errno;
 
 
int main(int argc, char**argv)
{
    int sockfd  = socket(AF_INET,SOCK_DGRAM,0);
    struct sockaddr_in sin;
    memset(&sin,0,sizeof(sin));
    sin.sin_port = htons(PORT);
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = inet_addr("127.0.0.1");
    char send[100],recv[100];
    int n = 0;
    puts("begin connect");
    int r = connect(sockfd,(SA*)&sin,sizeof(sin));  // udp的连接在这一步不会有问题  与 tcp不同, tcp会3次握手, udp没有
    if(r < 0){
        perror("connect error");
        return 0;
    }
    printf("connect return :%d , errno:%d\n", r, errno);
 
    while(1){
        n = read(0,send,100);
        n = write(sockfd,send,n);    // 这里也不会有问题,仅仅发送到缓冲区.
        printf("sendto return : %d\n" , n);
 
        n = read(sockfd,recv,100);  // 这里将返回错误Connection refused,这是没connect前所没有的
        if(n < 0){
            printf("***recvfrom return :%d ,error:%d\n",n,errno);
            perror("***read error");
        }  else {
            recv[n]  =0;
            printf("buf : %s\n", recv);
        }
    }
 
 
 
    return 0;
}
```

