title: Bash 日期时间戳格式化
date: 2016-12-05 12:09:45
tags: [Bash,Linux]
toc: true
---
```
UDF - 2016-11-29 00:00:35 --> tdParams_to==>{"appid":"2005001xxx","eventtime":"1480348772878","spreadurl":"dMxpxx","appkey":"XXbbf2fbaeb64d7b9623ec4ee3cde575","devicetype":"iPhone7,2","adnetname":"UC","osversion":"10.1.1","idfa":"DE055F0E-72D7-4A72-B7A0-C3E45A3A33XX","clicktime":"1480348581122","ip":"117.38.163.147"}UDF - 2016-11-29 00:00:41 --> tdParams_to==>{"appid":"2005001xxx","eventtime":"1480348766255","appkey":"XXbbf2fbaeb64d7b9623ec4ee3cde575","devicetype":"iPhone7,2","osversion":"10.0.2","idfa":"93271665-1341-499C-BDD6-6797699B9AXX","ip":"58.59.64.234"}
```

有以上格式的数据，需要将其中的`eventtime`时间戳筛选出来并格式化

<!-- more -->
### 方法1——命令行

将数据保存到`1.txt`中，执行如下命令

```
awk '{print substr($0,80,10)}' 1.txt | xargs -I {} date -d @{}  "+%Y-%m-%d %H:%M:%S"
```

上述命令中`xargs`表示将之前的内容作为参数，`{}`为其参数别名，然后调用系统命令`date`将日期格式化

### 方法2——shell脚本

```
#!/bin/bash#grep "tdParam" $1 | while read line cat $1 | while read line do        eventtime=${line:79:10}        pos=$(echo $line | sed 's/\(.*clicktime\)\(.*\)/\1/')        p=$((${#pos} + 3))        clicktime=${line:p:10}        echo ${line:6:19}" "`date -d @${eventtime} "+%Y-%m-%d %H:%M:%S"`" "`date -d @${clicktime} "+%Y-%m-%d %H:%M:%S"`done
```

其中`pos`那一表示获取关键字`clicktime`在行中的位置
