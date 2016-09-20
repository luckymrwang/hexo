title: 'error: command ''gcc'' failed with exit status 1'
date: 2016-09-01 14:34:12
tags: [gcc]
---

在部署`Open-Falcon`，[安装绘图组件](http://book.open-falcon.org/zh/quick_install/graph_components.html) `Dashboard` 执行 `./env/bin/pip install -r pip_requirements.txt` 时出现如下错误：

	error: command 'gcc' failed with exit status 1
	
解决方式：

```
sudo yum -y install gcc gcc-c++ kernel-devel
sudo yum -y install python-devel libxslt-devel libffi-devel openssl-devel
pip install cryptography
```