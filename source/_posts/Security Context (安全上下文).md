title: 容器 Security Context (安全上下文)
date: 2020-12-04 23:08:51
tags: [Docker,Kubernetes]
---
用来限制容器对宿主节点的可访问范围，以避免容器非法操作宿主节点的系统级别的内容，使得节点的系统或者节点上其他容器组受到影响。

Kubernetes 提供了三种配置 Security Context 的方法：

* Container-level Security Context：仅应用到指定的容器
* Pod-level Security Context：应用到 Pod 内所有容器以及 Volume
* Pod Security Policies（PSP）：应用到集群内部所有 Pod 以及 Volume

<!-- more -->
## Container-level Security Context
容器的定义中包含 securityContext 字段。通过指定该字段，可以为容器设定安全相关的配置。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
```

### privileged

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-no-privileged
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
	  imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
```

创建该 Pod，并开始 Shell 进入容器

```
kubectl exec -it security-context-demo -- sh
```

查看所有设备

```
# ls /dev
core             full             null             pts              shm              stdin            termination-log  urandom
fd               mqueue           ptmx             random           stderr           stdout           tty              zero
```

修改 hostname

```
# hostname
security-context-demo
# sysctl kernel.hostname=test
sysctl: error setting key 'kernel.hostname': Read-only file system
```

添加 privileged 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-privileged
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
        privileged: true
```

创建该 Pod，并开始 Shell 进入容器，查看所有设备
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/999D5D68-2277-4422-9DE2-67C8A271CDC1%202.png)

修改 hostname

```
# hostname
security-context-demo-privileged
# sysctl kernel.hostname=test
kernel.hostname = test
# hostname
test
```

| 字段名                     |    字段类型  | 字段说明            |
|----------------------------|---------------------|----------------------------------------------|
| privileged                | boolean                | 以 privileged 模式运行容器。容器则被授权访问宿主上所有设备，此时容器中的进程本质上等价于宿主节点上的 root 用户。默认值为 false  |
| 用户 <br> runAsUser                       | integer                | 执行容器 entrypoint 进程的 UID。默认为镜像元数据中定义的用户（dockerfile 中通过 USER 指令指定）。*也可以在Pod的SecurityContext中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准* |
| 用户组 <br> runAsGroup                    | integer                | 执行容器 entrypoint 进程的 GID。默认为 docker 引擎的 GID 。*也可以在Pod的SecurityContext中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准*                |
| 非Root <br> runAsNonRoot                | boolean                | 如果为 true，则 kubernetes 在运行容器之前将执行检查，以确保容器进程不是以 root 用户（UID为0）运行，否则将不能启动容器；如果此字段不设置或者为 false，则不执行此检查。*也可以在Pod的SecurityContext中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准* |
| 文件系统root只读 <br> readOnlyRootFilesystem | boolean                | 该容器的文件系统根路径是否为只读。默认为 false                                   |
| 允许扩大特权 <br> allowPrivilegeEscalation   | boolean                | 该字段控制了进程是否可以获取比父进程更多的特权。直接作用是为容器进程设置 no_new_privs标记。当如下情况发生时，该字段始终为 true|
|capabilities| add: array <br> drop: array | 为容器进程 add/drop Linux capabilities。默认使用容器引擎的设定 |
| seLinuxOptions | |此字段设定的 SELinux 上下文将被应用到容器。如果不指定，容器引擎将为容器分配一个随机的 SELinux 上下文。*也可以在Pod的SecurityContext中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准* |

### runAsUser & runAsGroup

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-privileged
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsUser: 1000
    	runAsGroup: 2000
```

在配置文件中，runAsUser 字段指定容器内的进程都使用用户 ID 1000 来运行。runAsGroup 字段指定所有容器中的进程都以主组 ID 2000 来运行。 如果忽略此字段，则容器的主组 ID 将是 root（0）。 当 runAsGroup 被设置时，所有创建的文件也会划归用户 1000 和组 2000。

执行命令进入容器的命令行界面，并在命令行界面中查看所有的进程

```
/ $ ps
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 1h
    7 1000      0:00 sh
   12 1000      0:00 ps
/ $ id
uid=1000 gid=2000
```

宿主机查看进程，容器内的一个进程对应于宿主机器上的一个进程
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/EA9499DA-BEF4-40BD-B1C4-EDB5F6F26F4E%202.png)

在宿主机上的进程`31523`的`uid`是 1000，`gid`是 2000

```
# ps --pid 31523 -O uid,uname,gid,group,ppid
  PID   UID USER       GID GROUP     PPID S TTY          TIME COMMAND
31523  1000 test      2000 2000     31475 S ?        00:00:00 sleep 1h
```

查看该进程的父进程

```
# ps --pid 31475 -O uid,uname,gid,group,ppid
  PID   UID USER       GID GROUP     PPID S TTY          TIME COMMAND
31475     0 root         0 root     14021 S ?        00:00:00 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/9801cd6ed9d86325d26723da6e9702359a3f50baa6c3d10ebd50315344122e2
```

### runAsNonRoot

```yaml
kind: Pod
metadata:
  name: security-context-demo-nonroot-root
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsNonRoot: true
```

执行命令以创建 Pod，并验证是否运行

```
# kubectl get pod
NAME                                 READY   STATUS                       RESTARTS   AGE
security-context-demo-nonroot-root   0/1     CreateContainerConfigError   0          3s
```

查看详情

```
# kubectl describe pod security-context-demo-nonroot-root
```
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/02D34D7D-9910-4014-A7C2-4A8760E68240%202.png)

指定非root用户

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-nonroot-user
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
```

执行命令以创建 Pod，并验证是否运行

```
# kubectl apply -f security-demo-nonroot-user.yaml
pod/security-context-demo-nonroot-user create

# kubectl get pod
NAME                                 READY   STATUS    RESTARTS   AGE
security-context-demo-nonroot-user   1/1     Running   0          4s
```

### readOnlyRootFilesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-readonly
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      readOnlyRootFilesystem: true
```

执行命令以创建 Pod，并执行命令进入容器的命令行界面

```
# whoami
root
# id
uid=0(root) gid=0(root) groups=10(wheel)

# mkdir a
mkdir: can't create directory 'a': Read-only file system
# touch a.txt
touch: a.txt: Read-only file system
```

### allowPrivilegeEscalation

一般情况下，execve() 系统调用能够赋予新启动的进程其父进程没有的权限，最常见的例子就是通过 setuid 和 setgid 来设置程序进程的 uid 和 gid 以及文件的访问权限。可以直接通过 fork 来提升进程的权限。
为了解决这个问题，Linux 内核从 3.5 版本开始，引入了 no\_new\_privs 属性（实际上就是一个 bit，可以开启和关闭），提供给进程一种能够在 execve() 调用整个阶段都能持续有效且安全的方法。
s
开启了 no\_new\_privs 之后，execve 函数可以确保所有操作都必须调用 execve() 判断并赋予权限后才能被执行。这就确保了线程及子线程都无法获得额外的权限，因为无法执行 setuid 和 setgid，也不能设置文件的权限。
一旦当前线程的 no\_new\_privs 被置位后，不论通过 fork，clone 或 execve 生成的子线程都无法将该位清零。

>ruid(real user ID):
ruid可以理解为哪个用户执行了这个程序或者文件, ruid就是谁.
>euid(effective user ID):
当进程执行时间, 操作系统会对euid进行识别, 以此来判断到底用什么权限来执行这个进程.
>也就是说操作系统实际是通过判断进程的euid而不是ruid来给予其权限，只是在大多数情况下, euid和ruid是相等的setuid是类unix系统提供的一个标志位， 其实际意义是set一个process的euid为这个可执行文件或程序的拥有者(比如root)的uid， 也就是说当setuid位被设置之后， 当文件或程序(统称为executable)被执行时, 操作系统会赋予文件所有者的权限, 因为其euid是文件所有者的uid

#### 默认

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-nonroot-user-privs-default
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox:ping
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
```

执行命令以创建 Pod，并执行命令进入容器的命令行界面

```
# ls -l /bin/ping
-rwsr-xr-x    1 root     root       1145088 Oct 12 23:47 /bin/ping

# ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.099 ms
64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.048 ms
```

#### 关闭allowPrivilegeEscalation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-nonroot-user-privs-false
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox:ping
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
```

执行命令以创建 Pod，并执行命令进入容器的命令行界面

```
# ls -l /bin/ping
-rwsr-xr-x    1 root     root       1145088 Oct 12 23:47 /bin/ping
# ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
ping: permission denied (are you root?)
```

### **Linux Capabilities**

在 Linux 2.2 版本之前，当内核对进程进行权限验证的时候，Linux 将进程划分为两类：特权进程（UID=0，也就是超级用户）和非特权进程（UID!=0），特权进程拥有所有的内核权限，而非特权进程则根据进程凭证（effective UID, effective GID，supplementary group 等）进行权限检查。

比如以常用的 passwd 命令为例，修改用户密码需要具有 root 权限，而普通用户是没有这个权限的。但是实际上普通用户又可以修改自己的密码。在 Linux 的权限控制机制中，有一类比较特殊的权限设置，比如 SUID(Set User ID on execution)，允许用户以可执行文件的 owner 的权限来运行可执行文件。因为程序文件 /bin/passwd 被设置了 SUID 标识，所以普通用户在执行 passwd 命令时，进程是以 passwd 的所有者，也就是 root 用户的身份运行，从而就可以修改密码了。

```sh
# ll /etc/shadow
----------. 1 root root 708 11月 15 09:50 /etc/shadow
# ll /bin/passwd
-rwsr-xr-x. 1 root root 27832 6月  10 2014 /bin/passwd
# su test
$ whoami
test
$ passwd
```

但是使用 SUID 却带来了新的安全隐患，当我们运行设置了 SUID 的命令时，通常只是需要很小一部分的特权，但是 SUID 却给了它 root 具有的全部权限，一旦 被设置了 SUID 的命令出现漏洞，就很容易被利用了。
为此 Linux 引入了 Capabilities 机制来对 root 权限进行了更加细粒度的控制，实现按需进行授权，这样就大大减小了系统的安全隐患。

### 如何使用 Capabilities
可以通过 `getcap` 和 `setcap` 两条命令来分别查看和设置程序文件的 `capabilities`属性。比如当前用户是test，使用 getcap 命令查看 ping 命令目前具有的 capabilities：

```
$ ll /bin/ping
-rwxr-xr-x. 1 root root 66176 8月   4 2017 /bin/ping
$ getcap /bin/ping
/bin/ping = cap_net_admin,cap_net_raw+p
```

可以看到具有 `cap_net_admin` 这个属性，所以现在可以执行 ping 命令：

```
$ ping baidu.com
PING baidu.com (220.181.38.148) 56(84) bytes of data.
64 bytes from 220.181.38.148 (220.181.38.148): icmp_seq=1 ttl=128 time=34.2 ms
64 bytes from 220.181.38.148 (220.181.38.148): icmp_seq=2 ttl=128 time=42.5 ms
```

但是如果把命令的 capabilities 属性移除掉：

```
$ sudo setcap cap_net_admin,cap_net_raw-p /bin/ping
$ getcap /bin/ping
/bin/ping =
```

这个时候再执行 ping 命令可以已经没有权限了

```
$ ping baidu.com
ping: socket: 不允许的操作
```

因为 `ping` 命令在执行时需要访问网络，所需的 `capabilities `为 `cap_net_admin` 和  `cap_net_raw`，所以可以通过 setcap 命令可来添加它们：

```
$ sudo setcap cap_net_admin,cap_net_raw+p /bin/ping
$ getcap /bin/ping
/bin/ping = cap_net_admin,cap_net_raw+p
$ ping baidu.com
PING baidu.com (220.181.38.148) 56(84) bytes of data.
64 bytes from 220.181.38.148 (220.181.38.148): icmp_seq=1 ttl=128 time=26.9 ms
64 bytes from 220.181.38.148 (220.181.38.148): icmp_seq=2 ttl=128 time=35.3 ms
```

命令中的 p 表示 Permitted 集合，+ 号表示把指定的capabilities 添加到这些集合中，- 号表示从集合中移除。

#### 线程的 capabilities
每一个线程，具有 5 个 capabilities 集合，每一个集合使用 64 位掩码来表示，显示为 16 进制格式。这 5 个 capabilities 集合分别是：

- Permitted
- Effective
- Inheritable
- Bounding
- Ambient

每个集合中都包含零个或多个 capabilities。这5个集合的具体含义如下

##### Permitted
定义了线程能够使用的 capabilities 的上限。它并不使能线程的 capabilities，而是作为一个规定。也就是说，线程可以通过系统调用 capset() 来从 Effective 或 Inheritable 集合中添加或删除 capability，前提是添加或删除的 capability 必须包含在 Permitted 集合中（其中 Bounding 集合也会有影响，具体参考下文）。 如果某个线程想向 Inheritable 集合中添加或删除 capability，首先它的 Effective 集合中得包含 CAP_SETPCAP 这个 capabiliy。

##### Effective
内核检查线程是否可以进行特权操作时，检查的对象便是 Effective 集合。如之前所说，Permitted 集合定义了上限，线程可以删除 Effective 集合中的某 capability，随后在需要时，再从 Permitted 集合中恢复该 capability，以此达到临时禁用 capability 的功能。

##### Inheritable
当执行exec() 系统调用时，能够被新的可执行文件继承的 capabilities，被包含在 Inheritable 集合中。这里需要说明一下，包含在该集合中的 capabilities 并不会自动继承给新的可执行文件，即不会添加到新线程的 Effective 集合中，它只会影响新线程的 Permitted 集合。

##### Bounding
Bounding 集合是 Inheritable 集合的超集，如果某个 capability 不在 Bounding 集合中，即使它在 Permitted 集合中，该线程也不能将该 capability 添加到它的 Inheritable 集合中。

Bounding 集合的 capabilities 在执行 fork() 系统调用时会传递给子进程的 Bounding 集合，并且在执行 execve 系统调用后保持不变。

- 当线程运行时，不能向 Bounding 集合中添加 capabilities。
- 一旦某个 capability 被从 Bounding 集合中删除，便不能再添加回来。
- 将某个 capability 从 Bounding 集合中删除后，如果之前 Inherited 集合包含该 capability，将继续保留。但如果后续从 Inheritable 集合中删除了该 capability，便不能再添加回来。

##### Ambient
Linux 4.3 内核新增了一个 capabilities 集合叫 Ambient ，用来弥补 Inheritable 的不足。Ambient 具有如下特性：

- Permitted 和 Inheritable 未设置的 capabilities，Ambient 也不能设置。
- 当 Permitted 和 Inheritable 关闭某权限（比如 CAP_SYS_BOOT）后，Ambient 也随之关闭对应权限。这样就确保了降低权限后子进程也会降低权限。
- 非特权用户如果在 Permitted 集合中有一个 capability，那么可以添加到 Ambient 集合中，这样它的子进程便可以在 Ambient、Permitted 和 Effective 集合中获取这个 capability。

Ambient 的好处显而易见，举个例子，如果你将 CAP_NET_ADMIN 添加到当前进程的 Ambient 集合中，它便可以通过 fork() 和 execve() 调用 shell 脚本来执行网络管理任务，因为 CAP_NET_ADMIN 会自动继承下去。

##### 文件的 capabilities

文件的 capabilities 被保存在文件的扩展属性中。如果想修改这些属性，需要具有 CAP_SETFCAP 的 capability。文件与线程的 capabilities 共同决定了通过 execve() 运行该文件后的线程的 capabilities。

文件的 capabilities 功能，需要文件系统的支持。如果文件系统使用了 nouuid 选项进行挂载，那么文件的 capabilities 将会被忽略。

类似于线程的 capabilities，文件的 capabilities 包含了 3 个集合：

- Permitted
- Inheritable
- Effective

这3个集合的具体含义如下：

**Permitted**
这个集合中包含的 capabilities，在文件被执行时，会与线程的 Bounding 集合计算交集，然后添加到线程的 Permitted 集合中。

**Inheritable**
这个集合与线程的 Inheritable 集合的交集，会被添加到执行完 execve() 后的线程的 Permitted 集合中。

**Effective**
这不是一个集合，仅仅是一个标志位。如果设置开启，那么在执行完 execve() 后，线程 Permitted 集合中的 capabilities 会自动添加到它的 Effective 集合中。对于一些旧的可执行文件，由于其不会调用 capabilities 相关函数设置自身的 Effective 集合，所以可以将可执行文件的 Effective bit 开启，从而可以将 Permitted 集合中的 capabilities 自动添加到 Effective 集合中。

下面通过具体的计算公式，来说明执行 execve() 后 capabilities 是如何被确定的。

用 P 代表执行 execve() 前线程的 capabilities，P' 代表执行 execve() 后线程的 capabilities，F 代表可执行文件的 capabilities。那么：

>P'(ambient) = (file is privileged) ? 0 : P(ambient)
>P'(permitted) = (P(inheritable) & F(inheritable)) |
                      (F(permitted) & P(bounding))) | P'(ambient)
> P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)
> P'(inheritable) = P(inheritable) [i.e., unchanged]
> P'(bounding) = P(bounding) [i.e., unchanged]

- 如果用户是 root 用户，那么执行 execve() 后线程的 Ambient 集合是空集；如果是普通用户，那么执行 execve() 后线程的 Ambient 集合将会继承执行 execve() 前线程的 Ambient 集合。
- 执行 execve() 前线程的 Inheritable 集合与可执行文件的 Inheritable 集合取交集，会被添加到执行 execve() 后线程的 Permitted 集合；可执行文件的 capability bounding 集合与可执行文件的 Permitted 集合取交集，也会被添加到执行 execve() 后线程的 Permitted 集合；同时执行 execve() 后线程的 Ambient 集合中的 capabilities 会被自动添加到该线程的 Permitted 集合中。
- 如果可执行文件开启了 Effective 标志位，那么在执行完 execve() 后，线程 Permitted 集合中的 capabilities 会自动添加到它的 Effective 集合中。
- 执行 execve() 前线程的 Inheritable 集合会继承给执行 execve() 后线程的 Inheritable 集合。

可以通过下面的命名来查看当前进程的 capabilities 信息：

```
# cat /proc/30472/status
CapInh:	00000000a80425fb
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
```

可以使用 capsh 命令把它们转义为可读的格式，这样基本可以看出进程具有的 capabilities 了：

```
# capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

### Docker Container Capabilities

Docker 容器本质上就是一个进程，所以理论上容器就会和进程一样会有一些默认的开放权限，默认情况下 Docker 会删除必须的 capabilities 之外的所有 capabilities，因为在容器中经常会以 root 用户来运行，使用 capabilities 后，容器中使用的 root 用户权限就比平时在宿主机上使用的 root 用户权限要少很多了，这样即使出现了安全漏洞，也很难破坏或者获取宿主机的 root 权限。
不过在运行容器的时候可以通过指定 —privileded 参数来开启容器的超级权限。
但是如果确实需要一些特殊的权限，可以通过 —cap-add 和 —cap-drop 这两个参数来动态调整，可以最大限度地保证容器的使用安全。

#### LinuxCapability常量
Linux Capabilities 常量格式为 CAP_XXX。然而，在容器定义中添加或删除 Linux Capabilities 时，必须去除常量的前缀 CAP_。例如：向容器中添加 CAP_SYS_TIME 时，只需要填写 SYS_TIME。
下面是从  [capabilities man page](http://man7.org/linux/man-pages/man7/capabilities.7.html)  中摘取的 Capabilites 列表：

| Docker Capability | Linux Capability |	描述 |
| ---------- | ---------- |  ---------- |
| CHOWN| CAP_CHOWN	| 修改文件所有者的权限|
| DAC_OVERRIDE| CAP_DAC\_OVERRIDE	| 忽略文件的 DAC 访问限制	|
| KILL| CAP_KILL	| 允许对不属于自己的进程发送信号|
| SYS_CHROOT| CAP_SYS\_CHROOT	| 允许使用 chroot() 系统调用|
| NET_BIND\_SERVICE| CAP\_NET\_BIND_SERVICE	| 允许绑定到小于 1024 的端口|
| SETUID| CAP_SETUID	| 允许改变进程的 UID|
| SETGID| CAP_SETGID	| 允许改变进程的 GID|
| FSETID| CAP_FSETID	| 允许设置文件的 setuid 位	|
| FOWNER| CAP_FOWNER	| 忽略文件属主 ID 必须和进程用户 ID 相匹配的限制|
| SETFCAP| CAP_SETFCAP	| 允许为文件设置任意的 capabilities	|
| SETPCAP| CAP_SETPCAP	| 允许向其它进程转移能力以及删除其它进程的任意能力(只限init进程)|
| NET_RAW| CAP_NET\_RAW	| 允许使用原始套接字|
| MKNOD| CAP_MKNOD	| 允许使用 mknod() 系统调用	|
| AUDIT_WRITE| CAP_AUDIT\_WRITE	| 将记录写入内核审计日志 |
|  - | - |
| NET_ADMIN| CAP_NET\_ADMIN	| 允许执行网络管理任务	|
| AUDIT_CONTROL| CAP_AUDIT\_CONTROL	| 启用和禁用内核审计；改变审计过滤规则；检索审计状态和过滤规则	|
| AUDIT_READ| CAP_AUDIT\_READ	| 允许通过 multicast netlink 套接字读取审计日志|
| BLOCK_SUSPEND| CAP_BLOCK\_SUSPEND	| 使用可以阻止系统挂起的特性|
| DAC_READ\_SEARCH| CAP\_DAC\_READ\_SEARCH	| 忽略文件读及目录搜索的 DAC 访问限制|
| IPC_LOCK| CAP_IPC\_LOCK	| 允许锁定共享内存片段	|
| IPC_OWNER| CAP_IPC\_OWNER	| 忽略 IPC 所有权检查|
| LEASE| CAP_LEASE	| 允许修改文件锁的 FL_LEASE 标志	|
| LINUX_IMMUTABLE| CAP\_LINUX\_IMMUTABLE	| 允许修改文件的 IMMUTABLE 和 APPEND 属性标志	|
| MAC_ADMIN| CAP\_MAC\_ADMIN	| 允许 MAC 配置或状态更改	|
| MAC_OVERRIDE| CAP_MAC\_OVERRIDE	| 覆盖 MAC(Mandatory Access Control)	|
| NET_BROADCAST| CAP_NET\_BROADCAST	| 允许网络广播和多播访问|
| SYS_ADMIN| CAP_SYS_ADMIN	| 允许执行系统管理任务，如加载或卸载文件系统、设置磁盘配额等	|
| SYS_BOOT| CAP_SYS\_BOOT	| 允许重新启动系统|
| SYS_MODULE| CAP_SYS\_MODULE	| 允许插入和删除内核模块	|
| SYS_NICE| CAP_SYS\_NICE	| 允许提升优先级及设置其他进程的优先级|
| SYS_PACCT| CAP_SYS\_PACCT	| 允许执行进程的 BSD 式审计|
| SYS_PTRACE| CAP_SYS\_PTRACE	| 允许跟踪任何进程|
| SYS_RAWIO| CAP_SYS\_RAWIO	| 允许直接访问 /devport、/dev/mem、/dev/kmem 及原始块设备|
| SYS_RESOURCE| CAP_SYS\_RESOURCE	| 忽略资源限制|
| SYS_TIME| CAP_SYS\_TIME	| 允许改变系统时钟	|
| SYS_TTY\_CONFIG| CAP\_SYS\_TTY_CONFIG	| 允许配置 TTY 设备|
| SYSLOG| CAP_SYSLOG	| 允许使用 syslog() 系统调用	|
| WAKE_ALARM| CAP_WAKE\_ALARM	| 允许触发一些能唤醒系统的东西(比如 CLOCK_BOOTTIME_ALARM 计时器)|

1. 启动一个容器，设置文件所有者和文件关联组

```
# docker container run --rm -it alpine chown nobody /
```

2. drop 掉所有 capabilities，只增加 CAP_CHOWN capability

```
# docker container run --rm -it --cap-drop ALL --cap-add CHOWN alpine chown nobody /
```

3. drop 掉  CAP_CHOWN capability

```
# docker container run --rm -it --cap-drop CHOWN alpine chown nobody /
chown: /: Operation not permitted
```

4. 给一个非 root 用户 nobody 创建一个容器并增加  CAP_CHOWN capability

```
docker container run --rm -it --cap-add chown -u nobody alpine chown nobody /
chown: /: Operation not permitted
```
docker当前只支持root用户级别 add 和 drop capalibities ，非root用户不支持

##### Docker 背后的容器管理 —— Libcontainer
[Libcontainer](https://github.com/opencontainers/runc/tree/master/libcontainer) 是Docker中用于容器管理的包，它基于Go语言实现，通过管理namespaces、cgroups、capabilities以及文件系统来进行容器控制。

在Docker中，对容器管理的模块为execdriver，目前Docker支持的容器管理方式有两种，一种就是最初支持的LXC方式，另一种称为native，即使用Libcontainer 进行容器管理。Docker Deamon 启动过程中就会对 execdriver进行初始化，会根据驱动的名称选择使用的容器管理方式。虽然在execdriver中只有LXC和native两种选择，但是native（即Libcontainer）通过接口的方式定义了一系列容器管理的操作，包括处理容器的创建（Factory）、容器生命周期管理（Container）、进程生命周期管理（Process）等一系列接口。

##### 容器的默认权限

```go
// New returns the docker default configuration for libcontainer
func New() libcontainer.Config {
	container := &libcontainer.Config{
		Capabilities: []string{
			“CHOWN”,
			“DAC_OVERRIDE”,
			“FSETID”,
			“FOWNER”,
			“MKNOD”,
			“NET_RAW”,
			“SETGID”,
			“SETUID”,
			“SETFCAP”,
			“SETPCAP”,
			“NET_BIND_SERVICE”,
			“SYS_CHROOT”,
			“KILL”,
			“AUDIT_WRITE”,
		},
		Namespaces: map[string]bool{
			“NEWNS”:  true,
			“NEWUTS”: true,
			“NEWIPC”: true,
			“NEWPID”: true,
			“NEWNET”: true,
		},
		Cgroups: &cgroups.Cgroup{
			Parent:          “docker”,
			AllowAllDevices: false,
		},
		MountConfig: &libcontainer.MountConfig{},
	}

	if apparmor.IsEnabled() {
		container.AppArmorProfile = “docker-default”
	}

	return container
}
```


### Kubernetes 中为容器设置 Linux Capabilities
在容器定义的 securityContext 中添加 capabilities 字段，可以向容器添加或删除 Linux Capability。

```yaml
securityContext:
  capabilities:
    drop:
    - KILL
    add:
    - NET_BIND_SERVICE
```

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Successfully serving on port 80\n")
}

func main() {
    http.HandleFunc("/", handler)
    log.Fatal(http.ListenAndServe(":80", nil))
}
```

制作Dockfile

```dockerfile
FROM alpine:latest

RUN mkdir /app
COPY server /app/
WORKDIR /app
ENTRYPOINT ["./server"]
```

构建镜像

```
docker build -t alpine:server .
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: server
  labels:
    run: my-server
spec:
  containers:
  - name: sec-ctx-demo
    image: alpine:server
    imagePullPolicy: IfNotPresent
    ports:
      - containerPort: 80
```

执行命令以创建 Pod，并验证是否运行

```
# kubectl apply -f server.yaml
pod/server created

# kubectl expose pod/server
service/server exposed

# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   9d
server       ClusterIP   10.100.208.118   <none>        80/TCP    6s

# curl 10.100.208.118
Successfully serving on port 80
```

修改 capacbilities ，去掉 CAP_NET_BIND_SERVICE

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: net-bind-service
  labels:
    run: my-server
spec:
  containers:
  - name: sec-ctx-demo
    image: alpine:server
    imagePullPolicy: IfNotPresent
    ports:
      - containerPort: 80
    securityContext:
      capabilities:
        drop:
        - NET_BIND_SERVICE
```

执行命令以创建 Pod，并验证是否运行

```
# kubectl apply -f net-bind-service.yaml

# kubectl get pod 
NAME                               READY   STATUS             RESTARTS   AGE
net-bind-service                   0/1     CrashLoopBackOff   5          5m24s

# kubectl logs net-bind-service
2020/11/17 07:40:22 listen tcp :80: bind: permission denied
```


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: drop-chown
spec:
  containers:
  - name: sec-ctx-demo
    image: alpine:latest
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      capabilities:
        drop:
        - CHOWN
```

*docker container run —rm -it —cap-drop CHOWN alpine chown nobody /**

给非 root 用户添加 capabilities

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: add-chown
spec:
  containers:
  - name: sec-ctx-demo
    image: alpine:latest
    imagePullPolicy: IfNotPresent
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - CHOWN
```

执行命令以创建 Pod，并验证是否运行

```
# kubectl apply -f add-chown.yaml
pod/add-chown created

# kubectl exec -it add-chown -- chown nobody /home
chown: /home: Operation not permitted
```

在 Kubernetes 中通过 `sercurityContext.capabilities`进行配置容器的Capabilities，当然最终还是通过 Docker 的`libcontainer`去借助 `Linux kernel capabilities` 实现的权限管理。

### SELinux

> SELinux 是由美国国家安全局 (NSA) 开发的，由内核实现的MAC ( Mandatory Access Control，强制访问控制)，可以说SELinux就是一个MAC系统。  

- 传统的文件权限与帐号关系：自主式存取控制, DAC
> 系统的帐号主要分为系统管理员 (root) 与一般用户，而这两种身份能否使用系统上面的文件资源则与 rwx 的权限配置有关。 但是各种权限配置对 root 是无效的。因此，当某个程序想要对文件进行存取时， 系统就会根据该程序的拥有者/群组，并比对文件的权限，若通过权限检查，就可以存取该文件了。这种存取文件系统的方式被称为『自主式存取控制 (Discretionary Access Control, DAC)』，基本上，就是依据程序的拥有者与文件资源的 rwx 权限来决定有无存取的能力。   

不过这种 DAC 的存取控制有几个困扰，那就是：
	- root 具有最高的权限：如果某个程序被非法取得， 且该程序属於 root 的权限，那么这个程序就可以在系统上进行任何资源的存取！
	- 使用者可以取得程序来变更文件资源的存取权限：如果将某个目录的权限配置为 777 ，由於对任何人的权限会变成 rwx ，因此该目录就会被任何人所任意存取！
- 以政策守则订定特定程序读取特定文件：强制访问控制, MAC
> 可以针对特定的程序与特定的文件资源来进行权限的控管！ 也就是说，即使是 root ，那么在使用不同的程序时，所能取得的权限并不一定是 root ， 而得要看当时该程序的配置而定。针对控制的『主体』变成了『程序』而不是使用者。 此外，这个主体程序也不能任意使用系统文件资源，因为每个文件资源也有针对该主体程序配置可取用的权限！ 这样，控制项目就细很多，但整个系统程序那么多、文件那么多，一项一项控制会很麻烦，所以 SELinux 也提供一些默认的政策 (Policy) ，并在该政策内提供多个守则 (rule) ，让你可以选择是否激活该控制守则。  
比如 httpd 这个程序， 默认情况下， httpd 仅能在 /var/www/ 这个目录底下存取文件，如果 httpd 这个程序想要到其他目录去存取数据时， 除了守则配置要开放外，目标目录也得要配置成 httpd 可读取的模式 (type) 才行，限制非常多， 所以，即使不小心 httpd 被 cracker 取得了控制权，他也无权浏览 /etc/shadow 等重要的配置。

#### SELinux 的三种模式

- Enforcing :  SELinux策略被强制执行，根据SELinux策略来拒绝或者是通过操作。
- Permissive:  SELinux策略并不会执行，原本在Enforcing式下应该被拒绝的操作，在该模式下只会触发安全事件日志记录，而不会拒绝此操作的执行。
- Disabled:  SELinux被关闭，SELinux不会执行任何策略

#### SELinux的启动和关闭

查看当前模式

```
# getenforce
Enforcing
```

修改模式配置

```
# cat /etc/selinux/config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

如果改变了政策则需要重新启动：如果由 enforcing 或 permissive 改成 disabled ，或由 disabled 改成其他两个，那也必须要重新启动。这是因为 SELinux 是整合到核心里面去的， 只可以在 SELinux 运行下切换成为强制 (enforcing) 或宽容 (permissive) 模式，不能够直接关闭 SELinux 。

```
# setenforce [0|1]
选项与参数：
0 ：转成 permissive 宽容模式；
1 ：转成 Enforcing 强制模式

# setenforce 0
# getenforce
Permissive
# setenforce 1
# getenforce
Enforcing
```

#### Security Context

```
# ll -Z
-rw-------. root root system_u:object_r:admin_home_t:s0 anaconda-ks.cfg
drwxr-xr-x. root root unconfined_u:object_r:admin_home_t:s0 security-demos
```

SELinux为每一个进程设置一个标签，称为进程的域，为文件设置标签，称为类型。每一标签由User、 Role、Type和Level这4部分组成。

- User: SELinux用户是由权限构成的集合，而非Linux用户。系统在登录的时候会为Linux用户匹配一个SELinux用户，通过semanage login -l 可以看到Linux用户和SELinux用户的映射。

```
# semanage login -l

登录名                  SELinux 用户           MLS/MCS 范围           服务

__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
system_u             system_u             s0-s0:c0.c1023       *
```

- Role:角色是一些类型的组合，是用户和类型的过渡。一个用户可以有多个角色。一个角色可以使用不同类型。
通过这个栏位，可以知道这个数据是属于程序、文件资源还是代表使用者。一般的角色有：
	* object_r：代表的是文件或目录等文件资源；
	* system_r：代表的就是程序，不过，一般使用者也会被指定成为 system_r

- Type: Type是SELinux访问控制的基础，描述进程所能访问的资源类型。常见文件资源的类型有blk_file, che_file, dir, fd, fifo_file, fie, filesystem, lnk_file和sock_file等，它们分别用于标识块文件、字符文件、目录、文件描述符文件、fifo文件、文件系统、链接和套接字文件等。容器文件一般表示为svirt_sandbox_file_t或者svirt_lxc_file_t。常见域有很多，不同类型的容器的需求不一样，容器域一般使用svirt_lxc_net_t来标示其域，也可以自己为容器定义域。
- Level:定义更加具体的权限，可以有两种选择，一种是MLS(多层级安全)，另外一种是MCS(多级分类安全)。

#### K8s中配置 SELinux

```yaml
seLinuxOptions:
  user: system_u
  role: system_r
  type: container_t
  level: s0:c829,c861
```

## Pod-level Security Context
在 Pod 的定义中增加 securityContext 字段。通过该字段指定的内容将对该 Pod 中所有的容器生效，并且还会影响Volume（包括 fsGroup 和 selinuxOptions ）

1、示例
￼
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/28F716FC-4DF2-49F0-AE08-F25375E2D964.png)

在上面的例子中：

	- spec.securityContext.runAsUser 字段指定了该 Pod 中所有容器的进程都以 UserID 1000 的身份运行，spec.securityContext.runAsGroup 字段指定了该 Pod 中所有容器的进程都以 GroupID 3000 的身份运行
		- 如果该字段被省略，容器进程的GroupID为 root(0)
		- 容器中创建的文件，其所有者为 userID 1000，groupID 3000
	- spec.securityContext.fsGroup 字段指定了该 Pod 的 fsGroup 为 2000
		- 数据卷 （本例中，对应挂载点 /data/demo 的数据卷为 sec-ctx-demo） 的所有者以及在该数据卷下创建的任何文件，其 GroupID 为 2000

2、创建 Pod 并进入容器执行 ps aux 查看所有进程，所有的进程都以 user 1000 的身份运行（由 runAsUser 指定），输出结果如下所示：
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/CB34CAFD-2CB6-432E-8512-E796071E644E.png)

3、执行 cd /data 切换到目录 /data，并查看目录中的文件列表 ls -l

/data/demo 目录的 groupID 为 2000（由 fsGroup 指定），输出结果如下所示：
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/A93CD144-C683-48AB-99DC-4DC4F4012015.png)

4、在命令行界面中，切换到目录 /data/demo，并创建一个文件 echo hello > testfile

testfile 的 groupID 为 2000 （由 FSGroup 指定），输出结果如下所示：
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/E5F1DA0F-F850-42E7-9C59-6390895923DC.png)

5、在命令行界面中执行 id 命令，输出结果如下所示：
![](Security%20Context%20(%E5%AE%89%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87)/B48414E4-B416-4F33-A562-E7F29E88A76A.png)

gid 为 3000，与 runAsGroup 字段所指定的一致
如果 runAsGroup 字段被省略，则 gid 取值为 0（即 root），此时容器中的进程将可以操作 root Group 的文件
可以设置的安全策略类型如下：

| 控制字段 | 字段名称 |
| ------ | ------ |
| 非Root <br> runAsNonRoot	 | 如果为 true，则 kubernetes 在运行容器之前将执行检查，以确保容器进程不是以 root 用户（UID为0）运行，否则将不能启动容器；如果此字段不设置或者为 false，则不执行此检查。该字段也可以在容器的 securityContext 中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准。|
| 用户 <br> runAsUser	| 执行容器 entrypoint 进程的 UID。默认为镜像元数据中定义的用户（dockerfile 中通过 USER 指令指定）。该字段也可以在容器的 securityContext 中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准。|
| 用户组 <br> runAsGroup |	执行容器 entrypoint 进程的 GID。默认为 docker 引擎的 GID。该字段也可以在容器的 securityContext 中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准。|
| fsGroup |	一个特殊的补充用户组，将被应用到 Pod 中所有容器。某些类型的数据卷允许 kubelet 修改数据卷的 ownership：<br>1. 修改后的 GID 取值来自于 fsGroup <br>2. setgid 标记位被设为 1（此时，数据卷中新创建的文件 owner 为 fsGroup）<br>3. permission 标记将与 rw-rw---- 执行或运算 <br>如果该字段不设置，kubelete 将不会修改数据卷的 ownership 和 permission |
| 补充用户组supplementalGroups	| 该列表中的用户组将被作为容器的主 GID 的补充，添加到 Pod 中容器的 enrtypoint 进程。可以不设置。|
| seLinuxOptions |	此字段设定的 SELinux 上下文将被应用到 Pod 中所有容器。如果不指定，容器引擎将为每个容器分配一个随机的 SELinux 上下文。该字段也可以在容器的 securityContext 中设定，如果 Pod 和容器的 securityContext 中都设定了这个字段，则对该容器来说以容器中的设置为准。|
| sysctls |	该列表中的所有 sysctl 将被应用到 Pod 中的容器。如果定义了容器引擎不支持的 sysctl，Pod 启动将会失败|

### 关于存储卷
Pod 的 securityContext 作用于 Pod 中所有的容器，同时对 Pod 的数据卷也同样生效。具体来说，fsGroup 和 seLinuxOptions 将被按照如下方式应用到 Pod 中的数据卷：

	- fsGroup：对于支持 ownership 管理的数据卷，通过 fsGroup 指定的 GID 将被设置为该数据卷的 owner，并且可被 fsGroup 写入。更多细节请参考 Ownership Management design document
	- seLinuxOptions：对于支持 SELinux 标签的数据卷，将按照 seLinuxOptions 的设定重新打标签，以使 Pod 可以访问数据卷内容。通常只需要设置 seLinuxOptions 中 level 这一部分内容。该设定为 Pod 中所有容器及数据卷设置 Multi-Category Security (MCS) 标签。

### Pod Security Policies
Pod Security Policies（PSP）是集群级的 Pod 安全策略，自动为集群内的 Pod 和 Volume 设置Security Context。
使用 PSP 需要在 kube-apiserver 服务的启动参数 --enable-admission-plugins 中进行设置：

--enable-admission-plugins=PodSecurityPolicy

在开启 PodSecurityPolicy 准入控制器后， Kubernetes 默认不允许创建任何 Pod，需要创建 PodSecurityPolicy 策略和相应的 RBAC 授权策略（Authorizing Policies），Pod 才能创建成功。

PodSecurityPolicy 对象定义了一组条件，指示 Pod 必须按系统所能接受的顺序运行。 它们允许管理员控制如下方面：

| 控制字段 |	字段名称 |
| ------- | ------- |
| 已授权容器的运行 |	privileged |
| 为容器添加默认的一组能力 |	defaultAddCapabilities |
| 为容器去掉某些能力 |	requiredDropCapabilities |
| 容器能够请求添加某些能力 |	allowedCapabilities |
| 控制卷类型的使用 |	volumes |
| 主机网络的使用 |	hostNetwork |
| 主机端口的使用 |	hostPorts |
| 主机 | PID namespace 的使用	hostPID |
| 主机 | IPC namespace 的使用	hostIPC |
| 主机路径的使用 |	allowedHostPaths |
| 容器的 | SELinux 上下文	seLinux |
| 用户 | ID	runAsUser |
| 配置允许的补充组 |	supplementalGroups |
| 分配拥有 | Pod 数据卷的 FSGroup	fsGroup |
| 必须使用一个只读的 | root 文件系统	readOnlyRootFilesystem |