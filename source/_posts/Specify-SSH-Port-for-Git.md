title: Specify SSH Port for Git
date: 2019-03-14 18:25:31
tags: [Git]
---

当Git服务器的SSH端口不是默认22时，可以修改`~/.ssh/config`来连接

```js
Host github.com
Port 22

Host *
User git
Port 1234
```