title: Kubernetes Volume 挂载
date: 2020-08-04 14:15:49
tags: [Kubernetes]
---

### 挂载配置文件到应用程序所在的目录

应用程序代码：
 
```go
package main

import (
    "fmt"
    "io"
    "io/ioutil"
    "log"
    "net/http"

    "gopkg.in/yaml.v2"
)

type config struct {
    Port string `yaml:"port"`
}

var c = new(config)

func init() {
    fContent, err := ioutil.ReadFile("config.yaml")
    if err != nil {
        log.Fatal(err)
    }

    if err = yaml.Unmarshal(fContent, c); err != nil {
        log.Fatal(err)
    }
}

func main() {
    handlerFunc := func(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "Hello !")
    }

    http.HandleFunc("/", handlerFunc)
    log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", c.Port), nil))
}
```
<!-- more -->

配置文件

```
port:8080
```
Dockerfile

```docker
FROM centos
COPY server /home/server
# 工作目录一定要指定，因为代码里读 config.yaml 写的是相对路径
WORKDIR /home
CMD /home/server
```

然后开始编写k8s部署的yaml文件

```yaml
# httpserver 依赖的配置文件
apiVersion: v1
data:
  config.yaml: |
    port: 8080
kind: ConfigMap
metadata:
  name: httpserver-config
  namespace: default

---

# httpserver deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpserver
  labels:
    app: httpserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpserver
  template:
    metadata:
      labels:
        app: httpserver
    spec:
      volumes:
      - name: config
        configMap: 
          name: httpserver-config
      containers:
      - name: bookstore
        image: uhub.service.ucloud.cn/wangkai/httpserver:v0.0.1
        ports:
        - containerPort: 8080
        # 把配置文件挂载到 /home 目录去
        volumeMounts:
        - name: config
          mountPath: /home
```

执行这些 YAML 配置后，我们会发现程序没有启动起来，报错：`/bin/sh: /home/server: No such file or directory`。原因很简单，我们把 config volume 挂载到 /home 目录后覆盖了该目录下的文件。以至于此时 /home 目录下只有 config.yaml，原先的二进制文件被覆盖掉了。

解决的办法是：

- 把配置文件挂载到其他目录，比如 /data，然后修改应用程序代码，去 /data 目录读。
- 添加 subPath 配置，subPath 可以指明使用 volume 的一个子目录，而不是其整个根目录。

第一种办法曲线救国，我们使用第二种 k8s 自身的解决方案来解决问题，只需要修改几行配置即可。

```yaml
spec:
  volumes:
  - name: config
    configMap: 
      name: httpserver-config
  containers:
  - name: bookstore
    image: uhub.service.ucloud.cn/wangkai/httpserver:v0.0.1
    ports:
    - containerPort: 9090
    volumeMounts:
    - name: config
      # 在目录地址后加上文件名，与 subPath 中指定的文件名相同
      mountPath: /home/config.yaml
      # 使用 config volume 的 config.yaml 文件，而不是整个 volume
      subPath: config.yaml
```

修改后再执行 kubectl apply -f xx.yaml 就可以运行了，describe Pod 查看，能看到挂载情况：

```
Mounts:
  /home/config.yaml from config (rw,path="config.yaml")
  /var/run/secrets/kubernetes.io/serviceaccount from default-token-jp596 (ro)
```

### 同时挂 ConfigMap & Secret 到同一目录下

有些场景，我们的配置文件可能不止一个，我们的应用程序要读取当前目录下的多个配置文件，比如既有 ConfigMap 也有 Secret。Docker 是不允许多个 Volume 挂到同一目录的，此类情况也可以通过 subPath 得到解决

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpserver
  labels:
    app: httpserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpserver
  template:
    metadata:
      labels:
        app: httpserver
    spec:
      volumes:
      - name: db-secret
        secret:
          secretName: db-secret
      - name: config
        configMap: 
          name: httpserver-config
      containers:
      - name: httpserver
        image: uhub.service.ucloud.cn/wangkai/httpserver:v0.0.1
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /home/config.yaml
          # 只挂载 volume 的 config.yaml 而不是整个 volume
          subPath: config.yaml
        - name: db-secret
          mountPath: /home/secret.yaml
          # 只挂载 volume 的 secret.yaml 而不是整个 volume
          subPath: secret.yaml
```


