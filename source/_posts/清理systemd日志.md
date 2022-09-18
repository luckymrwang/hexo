title: 清理systemd日志
date: 2022-09-04 22:15:22
tags: [Linux]
---
`systemd journal` 之于 `systemd` 犹如 `syslog` 之于 `init`，其日志文件保存在 `/var/log/journal` 目录下。随着时间的流逝，该目录下会积累大量日志文件，占用不少的磁盘空间。如果硬盘容量较小或可用空间紧张，可以考虑清理过期日志释放占用的空间。

本文介绍清理systemd日志的方法。

<!-- more -->
开启 `kubelet` 详情日志：

在 `/var/lib/kubelet/kubeadm-flags.env`  文件追加 `--v=4`

```sh
KUBELET_KUBEADM_ARGS=--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --v=4
```

重启 kubelet 服务：

```sh
systemctl restart kubelet
```

### 清理systemd日志

清理之前，可查看一下 `systemd` 日志所占用的磁盘空间。既可以用常用的 `du` 命令：

```sh
sudo du -sh /var/log/journal/
# 示例输出
# 3.9G /var/log/journal/
```

但更推荐使用systemd日志管理专用命令 `journalctl`：

```sh
journalctl --disk-usage
# 示例输出
# Archived and active journals take up 3.9G on disk.
```

知道了日志占用的磁盘空间，接下来便可以清理过期日志。开始之前，建议 `rotate` 当前日志(rotate是日志操作中的一个术语，其归档旧日志，后续日志写入新创建的日志文件中)：

```sh
sudo journalctl --rotate
```

`journalctl` 提供了三种清理 `systemd` 日志的方式。第一种是清理指定时间之前的日志：

```sh
# 清理7天之前的日志
sudo journalctl --vacuum-time=7d
# 清理2小时之前的日志
sudo journactl --vacuum-time=2h
# 清理10秒之前的日志
sudo journalctl --vacuum-time=10s
# 上述命令示例输出：
# Vacuuming done, freed 3.7G of archived journals on disk.
```

第二种是限制日志占用的空间大小：

```sh
# 限制systemd日志占用不超过1G空间
sudo journalctl --vacuum-size=1G
# 限制systemd日志占用不超过100M
sudo journalctl --vacuum-size=100M
# 输出与第一种类似
```

第三种是保留日志文件个数：

```sh
# 保留最近的5个日志文件
sudo journalctl --vacuum-files=5
# 输出与第一种类似
```

不知道 `journalctl` 管理日志功能之前，本人用过 find 配合 exec (或者管道加xargs)的土办法清理过期日志：

```sh
# 删除7天前的日志
find /var/log/journal -mtime +7 -exec rm -rf {} \;
```

### 一劳永逸的办法

上文介绍的清理systemd日志方法适合一次性手动管理，重复做就没意思了。一劳永逸的办法是配置systemd journal，让其自动管理日志，不占用过多磁盘空间。

方法是编辑 `/etc/systemd/journald.conf` 文件，对其中的参数进行设置。例如限制日志最大占用1G空间：

```sh
[Journal]
#Storage=auto
#Compress=yes
#Seal=yes
#SplitMode=uid
#SyncIntervalSec=5m
#RateLimitInterval=30s
#RateLimitBurst=1000
SystemMaxUse=1G
#SystemKeepFree=
#SystemMaxFileSize=
#RuntimeMaxUse=
#RuntimeKeepFree=
#RuntimeMaxFileSize=
```

保存配置文件后记得重新加载：`sudo systemctl restart systemd-journald`

### 参考

- [How to Clear Systemd Journal Logs](https://linuxhandbook.com/clear-systemd-journal-logs/)
- [Linux文件查找](https://itlanyan.com/find-files-in-linux/)

