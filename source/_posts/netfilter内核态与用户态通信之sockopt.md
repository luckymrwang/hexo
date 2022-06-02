title: netfilter内核态与用户态通信之sockopt
date: 2022-05-31 19:12:51
tags: [Socket]
---
用户态与内核态交互通信的方法不止一种，sockopt是比较方便的一个，写法也简单.

缺点就是使用 copy_from_user()/copy_to_user()完成内核和用户的通信， 效率其实不高， 多用在传递控制 选项 信息，不适合做大量的数据传输。

用户态函数：

- 发送：int setsockopt ( int sockfd, int proto, int cmd, void *data, int datelen);
- 接收：int getsockopt(int sockfd, int proto, int cmd, void *data, int datalen)
	- 第一个参数是socket描述符；
	- 第二个参数proto是sock协议，IP RAW的就用SOL_SOCKET/SOL_IP等，TCP/UDP socket的可用SOL_SOCKET/SOL_IP/SOL_TCP/SOL_UDP等，即高层的socket是都可以使用低层socket的命令字 的，IPPROTO_IP；
	- 第三个参数cmd是操作命令字，由自己定义；
	- 第四个参数是数据缓冲区起始位置指针，set操作时是将缓冲区数据写入内核，get的时候是将内核中的数 据读入该缓冲区；
	- 第五个参数数据长度

<!-- more -->
内核态函数：

- 注册：nf_register_sockopt(struct nf_sockopt_ops *sockops)
- 解除：nf_unregister_sockopt(struct nf_sockopt_ops *sockops)

结构体 nf_sockopt_ops test_sockops

```c++
struct nf_sockopt_ops {
    struct list_head list;

    u_int8_t pf; // 协议族

    /* Non-inclusive ranges: use 0/0/NULL to never get called. */
    int set_optmin;// 定义最小set命令字
    int set_optmax;// 定义最大set命令字
    int (*set)(struct sock *sk, int optval, void __user *user, unsigned int len);// 定义set处理函数
#ifdef CONFIG_COMPAT
    int (*compat_set)(struct sock *sk, int optval,
            void __user *user, unsigned int len);
#endif
    int get_optmin;// 定义最小get命令字
    int get_optmax;// 定义最大get命令字
    int (*get)(struct sock *sk, int optval, void __user *user, int *len);// 定义get处理函数
#ifdef CONFIG_COMPAT
    int (*compat_get)(struct sock *sk, int optval,
            void __user *user, int *len);
#endif
    /* Use the module struct to lock set/get code in place */
    struct module *owner;
};
```

其中命令字不能和内核已有的重复，宜大不宜小。命令字很重要，是用来做标识符的。而且用户态和内核态要定义的相同，

系统调用如下：sockt==》sock_common_setsockopt==》sk->sk_prot->setsockopt==》ip_setsockopt（tcp为例）

　　　　==>do_ip_setsockopt

　　　　==> nf_setsockopt==>nf_sockopt


```c++
int ip_setsockopt(struct sock *sk, int level,
        int optname, char __user *optval, unsigned int optlen)
{
    int err;

    if (level != SOL_IP)
        return -ENOPROTOOPT;

    err = do_ip_setsockopt(sk, level, optname, optval, optlen);
#ifdef CONFIG_NETFILTER
    /* we need to exclude all possible ENOPROTOOPTs except default case */
    if (err == -ENOPROTOOPT && optname != IP_HDRINCL &&
            optname != IP_IPSEC_POLICY &&
            optname != IP_XFRM_POLICY &&
            !ip_mroute_opt(optname)) {
        lock_sock(sk);
        err = nf_setsockopt(sk, PF_INET, optname, optval, optlen);
        release_sock(sk);
    }
#endif
    return

 err;
}

/* Call get/setsockopt() */
static int nf_sockopt(struct sock *sk, u_int8_t pf, int val,
              char __user *opt, int *len, int get)
{
    struct nf_sockopt_ops *ops;
    int ret;

    ops = nf_sockopt_find(sk, pf, val, get);
    if (IS_ERR(ops))
        return PTR_ERR(ops);

    if (get)
        ret = ops->get(sk, val, opt, len);
    else
        ret = ops->set(sk, val, opt, *len);

    module_put(ops->owner);
    return ret;
}

int nf_setsockopt(struct sock *sk, u_int8_t pf, int val, char __user *opt,
          unsigned int len)
{
    return nf_sockopt(sk, pf, val, opt, &len, 0);
}
```

![image](/images/netfilter1.png)

### 内核态的module.c

```c++
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <linux/string.h>
#include <linux/netfilter_ipv4.h>
#include <linux/init.h>
#include <asm/uaccess.h> 
 
#define SOCKET_OPS_BASE          128
#define SOCKET_OPS_SET       (SOCKET_OPS_BASE)
#define SOCKET_OPS_GET      (SOCKET_OPS_BASE)
#define SOCKET_OPS_MAX       (SOCKET_OPS_BASE + 1)
 
#define KMSG          "--------kernel---------"
#define KMSG_LEN      sizeof("--------kernel---------")
 
MODULE_LICENSE("GPL");
MODULE_AUTHOR("SiasJack");/*作者*/
MODULE_DESCRIPTION("sockopt module,simple module");//描述
MODULE_VERSION("1.0");//版本号
 
static int recv_msg(struct sock *sk, int cmd, void __user *user, unsigned int len)
{
    int ret = 0;
    printk(KERN_INFO "sockopt: recv_msg()\n"); 
 
    if (cmd == SOCKET_OPS_SET)
    {   
        char umsg[64];
        int len = sizeof(char)*64;
        memset(umsg, 0, len);
        ret = copy_from_user(umsg, user, len);
        printk("recv_msg: umsg = %s. ret = %d\n", umsg, ret);    
    }   
    return 0;
} 
 
static int send_msg(struct sock *sk, int cmd, void __user *user, int *len)
{
    int ret = 0;
    printk(KERN_INFO "sockopt: send_msg()\n"); 
    if (cmd == SOCKET_OPS_GET)
    {   
        ret = copy_to_user(user, KMSG, KMSG_LEN);
        printk("send_msg: umsg = %s. ret = %d. success\n", KMSG, ret);
    }   
    return 0;
 
}
 
static struct nf_sockopt_ops test_sockops =
{
    .pf = PF_INET,
    .set_optmin = SOCKET_OPS_SET,
    .set_optmax = SOCKET_OPS_MAX,
    .set = recv_msg,
    .get_optmin = SOCKET_OPS_GET,
    .get_optmax = SOCKET_OPS_MAX,
    .get = send_msg,
    .owner = THIS_MODULE,
};
 
static int __init init_sockopt(void)
{
    printk(KERN_INFO "sockopt: init_sockopt()\n");
    return nf_register_sockopt(&test_sockops);
}
 
static void __exit exit_sockopt(void)
{
    printk(KERN_INFO "sockopt: fini_sockopt()\n");
    nf_unregister_sockopt(&test_sockops);
}
 
module_init(init_sockopt);
module_exit(exit_sockopt);
```

### 用户态的user.c

```c++
#include <unistd.h>
#include <stdio.h>
#include <sys/socket.h>
#include <linux/in.h>
#include <string.h>
#include <errno.h> 
 
#define SOCKET_OPS_BASE      128
#define SOCKET_OPS_SET       (SOCKET_OPS_BASE)
#define SOCKET_OPS_GET      (SOCKET_OPS_BASE)
#define SOCKET_OPS_MAX       (SOCKET_OPS_BASE + 1) 
 
#define UMSG      "----------user------------"
#define UMSG_LEN  sizeof("----------user------------") 
 
char kmsg[64]; 
 
int main(void)
{
    int sockfd;
    int len;
    int ret; 
 
    sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
    if(sockfd < 0)
    {   
        printf("can not create a socket\n");
        return -1; 
    }   
 
    /*call function recv_msg()*/
    ret = setsockopt(sockfd, IPPROTO_IP, SOCKET_OPS_SET, UMSG, UMSG_LEN);
    printf("setsockopt: ret = %d. msg = %s\n", ret, UMSG);
    len = sizeof(char)*64; 
 
    /*call function send_msg()*/
    ret = getsockopt(sockfd, IPPROTO_IP, SOCKET_OPS_GET, kmsg, &len);
    printf("getsockopt: ret = %d. msg = %s\n", ret, kmsg);
    if (ret != 0)
    {   
        printf("getsockopt error: errno = %d, errstr = %s\n", errno, strerror(errno));
    }   
 
    close(sockfd);
    return 0;
}
```

### Makefile----系统不同命令可能不同，fedora 12

```makefile
TARGET = socketopt
OBJS = module.o
MDIR = drivers/misc
 
EXTRA_CFLAGS = -DEXPORT_SYMTAB
CURRENT = $(shell uname -r)
KDIR = /lib/modules/$(CURRENT)/build
PWD = $(shell pwd)
DEST = /lib/modules/$(CURRENT)/kernel/$(MDIR)
 
obj-m := $(TARGET).o
 
$(TARGET)-objs :=$(OBJS)
 
default:
    make -C  $(KDIR) SUBDIRS=$(PWD) modules 
    gcc -o user user.c
$(TARGET).o: $(OBJS)
    $(LD) $(LD_RFLAG) -r -o $@ $(OBJS)
 
insmod:
    insmod $(TARGET).ko
rmmod:
    rmmod $(TARGET).ko
 
clean:
    -rm -rf *.o *.ko .$(TARGET).ko.cmd .*.flags *.mod.c modules.order  Module.symvers .tmp_versions
    -rm -rf protocol/*.o protocol/.*.o.cmd *.markers 
    -rm -rf user
-include $(KDIR)/Rules.make
```

### 运行的结果

```sh
[root@root socket]# make   //编译
make -C  /lib/modules/2.6.31.5-127.fc12.i686.PAE/build SUBDIRS=/root/study/c_study/socket modules 
make[1]: Entering directory `/usr/src/kernels/2.6.31.5-127.fc12.i686.PAE'
  CC [M]  /root/study/c_study/socket/module.o
  LD [M]  /root/study/c_study/socket/socketopt.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /root/study/c_study/socket/socketopt.mod.o
  LD [M]  /root/study/c_study/socket/socketopt.ko
make[1]: Leaving directory `/usr/src/kernels/2.6.31.5-127.fc12.i686.PAE'
gcc -o user user.c
[root@root socket]# 
[root@root socket]# make insmod   //加载
insmod socketopt.ko
[root@root socket]# 
[root@root socket]# lsmod    //查看加载成功
Module                  Size  Used by
socketopt               1968  0 
sunrpc                158388  1 
[root@root socket]# dmesg -c  //清楚以前的系统信息
[root@root socket]# ./user  //运行用户态
setsockopt: ret = 0. msg = ----------user------------
getsockopt: ret = 0. msg = --------kernel---------
[root@root socket]# dmesg  //查看最新生成的日志
sockopt: recv_msg()
recv_msg: umsg = ----------user------------. ret = 0
sockopt: send_msg()
send_msg: umsg = --------kernel---------. ret = 0. success
```
