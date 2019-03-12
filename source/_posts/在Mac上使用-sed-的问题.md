title: 在Mac上使用 sed 的问题
date: 2019-03-12 14:19:04
tags: [Mac,Linux]
---

```
sed -i  's/hello/world/g' hello.php
```

上面这行代码，可以在 linux 上运行，作用是将找到的 hello 替换为 world，并且直接保存修改到文件。但是如果在 Mac 上，你会发现这行代码会报错。原因是在 Mac 上，sed 命令直接操作文件的时候，必须指定备份的格式，而在 linux 上，却并没有这个要求。

```
sed -i '' 's/hello/world/g' hello.php
```

如上面的代码所示，在 -i 之后加上一对引号，来指定备份格式，如果不需要备份，引号里的内容可以为空。