title: docker 搭建 gitlab-ce 教程
date: 2019-08-13 13:44:15
tags: [Docker]
---

### 获取镜像

```d
docker pull gitlab/gitlab-ce:12.1.4-ce.0
```

官方Hub镜像网络会比较慢，可以使用dotCloud加速器

```d
dao pull gitlab/gitlab-ce:12.1.4-ce.0
```

<!-- more -->
### 运行镜像

由于是docker镜像运行, 所以我们需要把gitlab的配置, 数据, 日志存到容器外面, 即将其挂载到宿主机。
先准备三个目录：

```sh
mkdir -p /Users/sino/Documents/luckymrwang/docker/gitlab/etc
mkdir -p /Users/sino/Documents/luckymrwang/docker/gitlab/logs
mkdir -p /Users/sino/Documents/luckymrwang/docker/gitlab/data
```

完整的运行命令如下：

```d
docker run -d \
    --hostname localhost.gitlab.com \  # 配置 /ect/hosts
    -p 80:80 \
    -p 443:443 \
    -p 22:22 \
    --name gitlab \
    --restart unless-stopped \
    -v /Users/sino/Documents/luckymrwang/docker/gitlab/etc:/etc/gitlab \
    -v /Users/sino/Documents/luckymrwang/docker/gitlab/logs:/var/log/gitlab \
    -v /Users/sino/Documents/luckymrwang/docker/gitlab/data:/var/opt/gitlab \
    -v /etc/localtime:/etc/localtime:ro \  # 保持宿主机和容器时间同步
    gitlab/gitlab-ce:12.1.4-ce.0
```

大约需要2分钟，然后运行 `docker ps` 看到状态显示为 'healthy' 就代表已经启动了

### 配置gitlab

要能充分使用gitlab, 必须配置邮件发送功能, 修改配置文件 gitlab.rb (启动镜像后产生的文件), 配置邮箱:

```d
vim /Users/sino/Documents/luckymrwang/docker/gitlab/etc/gitlab.rb
```

在文件的最后加上配置:

```d
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "xxx@qq.com"
gitlab_rails['smtp_password'] = "授权码"
gitlab_rails['smtp_domain'] = "smtp.qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = 'xxx@qq.com'
```

进入容器:

```d
docker exec -it gitlab /bin/bash
```

此时已经进入docker容器了, 容器中执行命令重新配置gitlab:

```d
gitlab-ctl reconfigure            # 重新配置
```

现在可以测试邮件是否配置正确了, 同样容器中执行:

```d
gitlab-rails console        # 进入邮件控制台, 稍等一会才能进入
Notify.test_email('xxx@qq.com', 'Message Subject', 'Message Body').deliver_now     # 发送测试邮件
```

现在邮件配置已经完成了, 需要配置项目路径 (如果你预留的gitlab映射端口是80的话, 到这里已经配置完了), 在宿主机中 (容器外面) 修改文件gitlab.yml, 如果host和port不对, 要改过来:

```d
vim /Users/sino/Documents/luckymrwang/docker/gitlab/data/gitlab-rails/etc/gitlab.yml
```

改完之后在容器中重启gitlab就配置完成了。 注意: 此时不能再重新配置(gitlab-ctl reconfigure), 否则可能会改变刚修改的gitlab.yml文件

```d
gitlab-ctl restart     # 重启gitlab
```

### gitlab常用命令

```d
# 重新应用gitlab的配置
gitlab-ctl reconfigure
 
# 重启gitlab服务
gitlab-ctl restart
 
# 查看gitlab运行状态
gitlab-ctl status
 
#停止gitlab服务
gitlab-ctl stop
 
# 查看gitlab运行日志
gitlab-ctl tail
```


