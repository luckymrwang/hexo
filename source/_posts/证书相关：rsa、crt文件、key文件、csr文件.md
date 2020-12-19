title: 证书相关：rsa、crt文件、key文件、csr文件
date: 2020-12-17 15:20:28
tags: [Linux]
---

- 非对称加密：一个公钥、一个私钥，公钥加密的文件可以用私钥解密，反之也可以；RSA就是一种常见的非对称加密算法；

另外，私钥一般自己保存，只有自己知道；公钥则是公开的

- openssl：一个开源的组织、一个开源的软件代码库和密码库工具，囊括了主要的密码算法；

<!-- more -->
怎么生成一个RSA的公钥和密钥对，并进行解密和加密？

```
openssl genpkey -algorithm rsa -out rsa_private.key
```

该命令生成了一个 *私钥公钥对*，也就是说rsa_private.key这个文件同时包含了私钥和公钥；

采用如下命令可以查看其中的公钥和密钥：

```
$ openssl rsa -in rsa_private.key -text
Private-Key: (2048 bit)
modulus:
    00:c0:85:23:50:15:35:1c:4d:5b:f9:7f:6c:cf:07:
    4e:7a:01:3d:d8:de:97:4f:3f:c6:11:5c:bb:2f:27:
    43:e6:2d:3e:ab:52:df:ba:8b:ea:f5:e3:89:ee:e8:
    87:82:76:ef:f1:72:87:5b:ec:02:6c:8e:18:39:95:
    a2:3c:48:f6:69:21:98:2a:69:5b:ca:f4:21:35:8e:
    85:2f:02:28:c5:08:94:02:8d:ee:e9:0f:11:b8:bb:
    fa:b2:57:87:42:92:b5:d2:57:7b:b2:a8:31:99:ad:
    de:72:1e:31:0d:5c:ac:ad:e9:01:08:f1:fe:1a:a2:
    36:f4:d2:7b:89:91:0e:88:a3:6e:3c:84:7d:32:c8:
    6a:64:db:27:87:8f:25:e6:fd:43:84:05:c9:95:4f:
    8a:4f:d0:8a:52:66:04:e5:24:81:77:c5:e4:5e:29:
    28:e1:df:bd:5e:ac:9a:52:e5:06:01:03:bb:e4:31:
    03:0e:3c:50:b7:a7:5e:bb:04:96:63:e6:bb:de:7d:
    85:a4:e7:35:dc:b2:f6:52:16:fc:e9:34:96:64:72:
    2c:1c:32:bb:9e:a3:b2:c2:64:bd:80:5e:52:6e:2c:
    c3:37:3c:b8:d0:a1:34:c0:da:cd:3e:ad:cc:56:57:
    24:33:d7:b3:2e:e1:30:47:b3:5b:ec:e3:5b:ea:06:
    86:9f
publicExponent: 65537 (0x10001)
privateExponent:
    00:95:e0:d0:9e:0e:f4:9b:05:0a:be:91:5a:57:4e:
    9b:e4:d5:c4:9d:6a:a5:27:78:41:ad:d0:a0:95:54:
    1f:43:3a:24:18:e2:da:f4:72:eb:47:e4:8d:c4:a5:
    d8:a1:54:10:f6:ca:af:e0:7b:3b:63:e1:b7:b0:54:
    f2:c9:b6:0f:c7:c6:f4:9c:c8:0b:43:54:8d:ea:10:
    fb:54:9e:7c:b8:f0:35:b2:4b:67:1c:9f:b3:af:3b:
    01:30:08:7e:6f:f0:a1:86:90:be:e7:56:93:ce:cd:
    92:69:0b:62:2a:c1:e4:59:3c:15:a7:2e:26:21:fb:
    f9:86:dd:ba:79:5d:a9:8f:eb:3a:42:39:0f:a9:9a:
    27:1e:ac:9a:fe:4c:69:7b:74:72:25:84:8e:3f:1c:
    86:27:aa:6c:93:0e:9d:55:1c:61:ad:d8:bf:01:35:
    b3:3b:ef:4f:70:f0:a8:dd:13:67:c7:af:58:77:42:
    80:de:52:03:d6:15:ad:25:6c:cc:d5:6d:f7:d2:c4:
    a5:cb:77:85:34:b4:8a:7a:4f:5e:de:9f:6f:59:ee:
    5e:cb:d1:60:01:aa:d3:90:4e:2d:53:c6:a9:35:1b:
    d7:04:0b:3a:6b:40:31:0b:f6:0a:57:54:c2:d4:6b:
    ec:6e:4e:17:5f:40:24:17:fb:cc:e7:e2:8f:f2:0b:
    45:31
prime1:
    00:df:27:0b:2b:3e:60:33:c6:6f:f4:d5:0e:7f:90:
    a2:e2:c4:d0:01:b9:f5:8a:93:cb:cf:df:c1:eb:23:
    5a:9b:49:f0:38:dc:04:b8:e4:61:db:38:83:95:87:
    80:a8:d6:92:09:61:c6:88:f0:13:60:d9:14:c4:03:
    0d:6b:2e:80:ec:19:4d:43:3b:08:b0:bd:6e:78:82:
    b4:2e:df:3d:b4:2b:39:d5:d8:eb:8a:a2:df:ae:fb:
    38:33:6f:f6:2f:fa:e0:f0:31:ee:93:1b:cd:35:ef:
    60:5b:c2:57:ee:37:d4:c2:c2:27:a9:21:61:40:69:
    ac:84:8d:a8:a2:1f:dc:33:07
```

可以通过如下的命令提取出其中的公钥：

```
$ openssl rsa -pubout -in rsa_private.key 
writing RSA key
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwIUjUBU1HE1b+X9szwdO
egE92N6XTz/GEVy7LydD5i0+q1Lfuovq9eOJ7uiHgnbv8XKHW+wCbI4YOZWiPEj2
aSGYKmlbyvQhNY6FLwIoxQiUAo3u6Q8RuLv6sleHQpK10ld7sqgxma3ech4xDVys
rekBCPH+GqI29NJ7iZEOiKNuPIR9MshqZNsnh48l5v1DhAXJlU+KT9CKUmYE5SSB
d8XkXiko4d+9XqyaUuUGAQO75DEDDjxQt6deuwSWY+a73n2FpOc13LL2Uhb86TSW
ZHIsHDK7nqOywmS9gF5SbizDNzy40KE0wNrNPq3MVlckM9ezLuEwR7Nb7ONb6gaG
nwIDAQAB
-----END PUBLIC KEY-----
 
 
$ openssl rsa -pubout -in rsa_private.key  -out rsa_pub.key
writing RSA key
```

现在我们有公钥和私钥了，怎么加密解密？

先生成一个测试文件：

```
$ echo "this is a test" > text
```

对该文件进行加密：

```
#采用公钥对文件进行加密
$ openssl rsautl -encrypt -in text -inkey rsa_pub.key -pubin -out text.en
 
#采用私钥解密文件
$ openssl rsautl -decrypt -in text.en -inkey rsa_private.key 
this is a test
```
既然是非对称加密，那我们尝试下用私钥加密，用公钥解密。

这里需要注意的是，私钥加密在openssl中对应的是-sign这个选项，公钥解密对应的是-verify这个选项，如下：

```
#用私钥对文件进行加密（签名）
$openssl rsautl -sign -in text -inkey rsa_private.key -out text.en
 
#用公钥对文件进行解密（校验）
$openssl rsautl -verify -in text.en -inkey rsa_pub.key -pubin
this is a test
```

以上大概介绍了公钥和私钥，那现在有一个问题：

公钥是公开分发的，那当你拿到一个公司（个人）的公钥之后，怎么确定这个公钥就是那个公司（个人）的？而不是一个别人篡改之后的公钥？而且公钥上没有任何的附加信息，标记当前公钥的所属的实体，相关信息等

为了解决这个问题，人们引入了如下两个概念：

（1）证书：公钥信息 + 额外的其他信息（比如所属的实体，采用的加密解密算法等）= 证书。证书文件的扩展名一般为crt。

拿到一个证书之后，可以通过openssl相关的命令来查看该证书的相关信息：

```

$ openssl x509 -in .minikube/apiserver.crt  -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2 (0x2)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=minikubeCA
        Validity
            Not Before: Feb 11 11:17:07 2019 GMT
            Not After : Feb 12 11:17:07 2020 GMT
        Subject: O=system:masters, CN=minikube
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:9f:6b:b8:4c:72:8c:e3:12:25:da:c2:1c:5f:28:
                    4a:bc:05:78:28:f3:70:4d:be:dd:61:23:bc:60:b9:
                    a8:43:3a:65:84:05:9f:38:b3:19:3c:b8:58:e6:57:
                    4d:b3:8e:3d:26:dc:c8:82:a2:65:1b:6f:48:e2:f9:
                    7d:69:77:c7:3d:1e:09:f0:4a:2f:a8:e0:bd:ca:42:
                    e9:a0:db:2c:7c:9b:c7:f4:a5:af:97:c6:4e:36:0f:
                    7b:7c:73:ff:05:80:8e:09:00:66:93:f4:c2:4d:a3:
                    d2:47:37:cf:db:e5:ba:cb:ee:10:23:ad:2f:29:87:
                    52:00:5a:f8:33:f3:6b:6c:4a:bf:86:ee:9f:5f:9e:
                    1c:65:f8:ac:45:02:cc:e8:e7:94:7e:51:92:9f:bb:
                    a8:ca:96:cc:67:91:82:65:c6:cd:61:bf:73:ec:74:
                    06:c1:53:15:bd:11:6b:49:8a:13:f6:e7:ad:da:49:
                    26:58:51:04:fb:53:2f:c7:6e:2d:90:be:d6:04:68:
                    99:79:d1:60:37:5f:5d:5a:08:ba:5a:79:18:5b:37:
                    ca:c2:fe:83:0d:e6:16:3d:fc:d2:b2:99:74:0d:86:
                    c7:55:08:bf:99:80:a9:a6:62:9e:1f:2b:89:25:1d:
                    b4:93:03:f6:d4:1d:39:37:ca:0b:15:03:fc:23:8b:
                    cd:fd
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Alternative Name: 
                DNS:minikubeCA, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes, DNS:localhost, IP Address:192.168.99.104, IP Address:10.96.0.1, IP Address:10.0.0.1
    Signature Algorithm: sha256WithRSAEncryption
         9d:17:e1:0d:d9:db:f1:60:90:fe:84:48:c7:7b:c9:a2:ec:f8:
         a8:18:95:5c:ec:dd:ef:02:5a:f7:2a:49:8a:68:0b:ed:8e:f5:
         8d:73:d9:64:d3:93:01:be:0d:08:62:d0:e8:3e:e6:3f:b9:17:
         0e:88:35:62:17:3b:65:42:01:bb:72:b2:c6:ac:1b:54:8d:08:
         1d:1a:02:2a:98:f1:f4:49:0c:50:92:fd:af:4f:12:03:3c:76:
         9d:08:ff:6f:9e:9a:60:25:96:89:91:1a:d3:23:78:cd:2b:84:
         c2:35:36:1e:de:0a:fe:2c:e3:2d:4b:ab:06:42:a8:ad:77:ec:
         d9:f2:a1:e4:00:18:6f:c6:33:08:b1:f8:8a:0a:d2:84:4a:b6:
         de:03:30:c0:6f:7f:0a:48:3e:74:be:56:3d:b9:f4:75:b2:19:
         86:4a:c4:cc:ae:a3:25:9a:7f:a2:8a:05:d5:0f:20:99:18:21:
         72:e0:4b:80:65:c6:ee:28:37:d5:d1:88:d5:c6:48:5a:d5:9c:
         a1:7d:d0:53:72:84:ce:95:83:56:9c:74:ec:f2:a3:c7:cc:27:
         20:b9:54:a7:a3:e1:f9:09:0e:14:dd:06:6a:3e:e0:37:5d:4e:
         10:1e:49:68:2b:cd:fc:c1:9b:4b:56:7e:7a:45:9a:3f:eb:09:
         e7:3f:c7:2b
```

另外，如果你使用的是mac系统，可以通过mac的钥匙链（keychain）打开正式，如：

![openssl](/images/openssl16.png)

（2）CA：证书认证中心；拿到一个证书之后，得先去找CA验证下，拿到的证书是否是一个“真”的证书，而不是一个篡改后的证书。如果确认证书没有问题，那么从证书中拿到公钥之后，就可以与对方进行安全的通信了，基于非对称加密机制。

CA自身的分发及安全保证，一般是通过一些权威的渠道进行的，比如操作系统会内置一些官方的CA、浏览器也会内置一些CA；

```
#采用CA校验一个证书
openssl verify -CAfile xxxx.crt usercert.crt
 
 
例如：
$ openssl verify -CAfile ca.cert tls.cert
 tls.cert: OK
```

那接下来的问题：

我想给自己，给公司、给我的某个服务器申请一个证书，该怎么搞？

公钥私钥对可以在自己的本地通过相关的工具（如openssl、ssh_keygen）产生，那公钥怎么包装成一个证书，并且要在CA那边“注册”一下，不然，别人拿到你的证书之后，去CA那边验证不过，会认为是一个不可信证书。

步骤如下：

（1）先生成一个公钥、密钥对

```
#生成一个公钥密钥对
openssl genpkey -algorithm rsa -out rsa_private.key
```

（2）基于该私钥我们生成一个CSR（证书签名请求）

```
#采用私钥生成一个CSR，过程中需要输入一些信息，这些信息都是公开的
$ openssl req -new -key rsa_private.key -out server.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:中国
string is too long, it needs to be less than  2 bytes long
Country Name (2 letter code) []:CN
State or Province Name (full name) []:BEIJING
Locality Name (eg, city) []:WANGJING
Organization Name (eg, company) []:xxxx
Organizational Unit Name (eg, section) []:wewe
Common Name (eg, fully qualified host name) []:lll
Email Address []:weiyuanke@xx.com
 
Please enter the following 'extra' attributes
to be sent with your certificate request
 
 
 
#CSR文件生成了，查看一下，可以看到我们输入的信息
$ openssl req -in server.csr -text -noout
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: C=CN, ST=BEIJING, L=WANGJING, O=xxxx, OU=wewe, CN=lll/emailAddress=weiyuanke@xx.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c0:85:23:50:15:35:1c:4d:5b:f9:7f:6c:cf:07:
                    4e:7a:01:3d:d8:de:97:4f:3f:c6:11:5c:bb:2f:27:
                    43:e6:2d:3e:ab:52:df:ba:8b:ea:f5:e3:89:ee:e8:
                    87:82:76:ef:f1:72:87:5b:ec:02:6c:8e:18:39:95:
                    a2:3c:48:f6:69:21:98:2a:69:5b:ca:f4:21:35:8e:
                    85:2f:02:28:c5:08:94:02:8d:ee:e9:0f:11:b8:bb:
                    fa:b2:57:87:42:92:b5:d2:57:7b:b2:a8:31:99:ad:
                    de:72:1e:31:0d:5c:ac:ad:e9:01:08:f1:fe:1a:a2:
                    36:f4:d2:7b:89:91:0e:88:a3:6e:3c:84:7d:32:c8:
                    6a:64:db:27:87:8f:25:e6:fd:43:84:05:c9:95:4f:
                    8a:4f:d0:8a:52:66:04:e5:24:81:77:c5:e4:5e:29:
                    28:e1:df:bd:5e:ac:9a:52:e5:06:01:03:bb:e4:31:
                    03:0e:3c:50:b7:a7:5e:bb:04:96:63:e6:bb:de:7d:
                    85:a4:e7:35:dc:b2:f6:52:16:fc:e9:34:96:64:72:
                    2c:1c:32:bb:9e:a3:b2:c2:64:bd:80:5e:52:6e:2c:
                    c3:37:3c:b8:d0:a1:34:c0:da:cd:3e:ad:cc:56:57:
                    24:33:d7:b3:2e:e1:30:47:b3:5b:ec:e3:5b:ea:06:
                    86:9f
                Exponent: 65537 (0x10001)
        Attributes:
            challengePassword        :unable to print attribute
    Signature Algorithm: sha256WithRSAEncryption
         30:55:9a:db:3e:a6:ba:99:d8:f0:6f:a9:26:bb:3e:d7:79:1a:
         ab:ee:99:7a:f5:eb:fa:49:cd:68:10:21:e6:08:a9:73:4e:af:
         5a:86:36:a4:8f:02:64:c4:9c:e3:54:0f:1a:56:c8:f3:29:94:
         82:cf:a7:da:7a:4b:2f:b3:70:d5:e7:7f:31:6d:0f:a0:9c:06:
         15:21:a3:52:66:7c:c0:d6:1d:fa:39:ae:4d:fb:91:d5:44:ea:
         96:6c:af:4e:d6:a8:10:92:c2:e1:9b:77:e7:f4:71:bb:78:64:
         71:16:01:be:c2:97:77:c6:99:b6:32:a7:e5:30:4d:9f:91:4c:
         9e:a3:4b:b8:d9:9e:55:ab:d0:ae:9c:9e:e6:ca:3f:ad:d1:fc:
         8a:a6:c8:7a:ec:d6:91:f1:93:5d:57:b9:07:e9:c7:3c:d4:d6:
         9b:a6:f3:75:b5:9a:d8:9f:4a:68:40:1c:6a:d8:17:50:81:ca:
         30:df:22:50:61:42:6a:6e:ee:12:40:71:63:74:76:55:58:1f:
         8e:75:5b:fd:79:0c:b9:fc:3d:ae:8f:d6:a9:5a:c7:bf:b7:20:
         29:d7:f1:5f:9f:20:ef:25:f4:05:a8:52:6c:9b:62:9b:3a:9e:
         4f:13:d5:c8:31:5a:b3:64:3f:01:91:5c:6e:46:61:f2:69:fe:
         00:7e:cb:24
```

（3）将该CSR文件发给CA，“注册一下”，当然了这个过程是收费的，要钱的。

这里，我们把自己当作一个CA，自己给自己注册一下，当然了，产生的证书是没人认可的。

```
#生成一个证书：mycert.crt 证书的有效期 365天
$ openssl x509 -req -days 365 -in server.csr -signkey rsa_private.key -out mycert.crt
Signature ok
subject=/C=CN/ST=BEIJING/L=WANGJING/O=xxxx/OU=wewe/CN=lll/emailAddress=weiyuanke@xx.com
Getting Private key
 
 
#查看证书的相关信息
 
$ openssl x509 -in mycert.crt -text
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 18029165557794453697 (0xfa34768d5d0bb8c1)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: C=CN, ST=BEIJING, L=WANGJING, O=xxxx, OU=wewe, CN=lll/emailAddress=weiyuanke@xx.com
        Validity
            Not Before: Feb 14 08:13:54 2019 GMT
            Not After : Feb 14 08:13:54 2020 GMT
        Subject: C=CN, ST=BEIJING, L=WANGJING, O=xxxx, OU=wewe, CN=lll/emailAddress=weiyuanke@xx.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c0:85:23:50:15:35:1c:4d:5b:f9:7f:6c:cf:07:
                    4e:7a:01:3d:d8:de:97:4f:3f:c6:11:5c:bb:2f:27:
                    43:e6:2d:3e:ab:52:df:ba:8b:ea:f5:e3:89:ee:e8:
                    87:82:76:ef:f1:72:87:5b:ec:02:6c:8e:18:39:95:
                    a2:3c:48:f6:69:21:98:2a:69:5b:ca:f4:21:35:8e:
                    85:2f:02:28:c5:08:94:02:8d:ee:e9:0f:11:b8:bb:
                    fa:b2:57:87:42:92:b5:d2:57:7b:b2:a8:31:99:ad:
                    de:72:1e:31:0d:5c:ac:ad:e9:01:08:f1:fe:1a:a2:
                    36:f4:d2:7b:89:91:0e:88:a3:6e:3c:84:7d:32:c8:
                    6a:64:db:27:87:8f:25:e6:fd:43:84:05:c9:95:4f:
                    8a:4f:d0:8a:52:66:04:e5:24:81:77:c5:e4:5e:29:
                    28:e1:df:bd:5e:ac:9a:52:e5:06:01:03:bb:e4:31:
                    03:0e:3c:50:b7:a7:5e:bb:04:96:63:e6:bb:de:7d:
                    85:a4:e7:35:dc:b2:f6:52:16:fc:e9:34:96:64:72:
                    2c:1c:32:bb:9e:a3:b2:c2:64:bd:80:5e:52:6e:2c:
                    c3:37:3c:b8:d0:a1:34:c0:da:cd:3e:ad:cc:56:57:
                    24:33:d7:b3:2e:e1:30:47:b3:5b:ec:e3:5b:ea:06:
                    86:9f
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha1WithRSAEncryption
         97:a8:05:57:46:81:e4:a9:e2:44:a6:85:71:f3:5b:2a:c2:7e:
         9c:70:e8:5e:ae:15:7f:ed:14:98:6e:52:4e:16:8d:89:70:7e:
         92:63:82:3c:e8:41:e6:b2:46:e1:b5:f8:5f:8d:c1:f8:71:1c:
         af:a5:30:56:1d:74:40:5a:55:6c:1a:74:8b:15:3b:9d:a2:d7:
         ff:1d:fa:1e:ad:0d:1d:bb:c0:42:17:65:25:74:d4:13:f1:e9:
         b1:26:64:e6:41:72:1c:13:b9:ff:8f:6a:f1:a7:e2:d2:b4:b8:
         85:37:62:2e:94:58:6e:2b:40:7c:c2:de:59:43:38:2b:35:29:
         47:1f:11:1b:65:b0:24:e0:7e:6c:3e:a4:47:17:ad:59:58:df:
         37:b7:66:4a:f9:6b:a1:ac:f7:ea:0e:c2:d5:1c:2d:0e:19:2f:
         6e:1d:a9:0a:06:a8:2c:2c:d8:01:65:a5:38:ea:3a:15:18:ed:
         f7:6b:f0:3c:a7:ed:0b:76:cb:57:ae:9c:a1:01:e7:28:90:e5:
         d4:d8:38:ed:6f:87:ca:26:7b:9d:74:fd:ba:e2:31:71:ab:ff:
         b4:07:a7:0d:f8:21:b7:84:19:1f:9a:13:e6:aa:88:17:32:48:
         79:4f:8e:ee:50:cf:ee:9d:bb:5d:77:1a:e2:96:67:d0:db:99:
         e4:7a:55:5b
```


