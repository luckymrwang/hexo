title: Centos下布置kale系统skyline
date: 2015-11-17 12:52:16
tags: [监控]
toc: true
---

[Skyline](https://github.com/etsy/skyline) 的页面上已经介绍了安装步骤，下面我记录一下在Centos上安装出现的问题已经相应的解决办法

Centos上默认安装了python2.6以及对应的pip，如果想升级[点击这里](http://ruiaylin.github.io/2014/12/12/python%20update/)

<!-- more -->
我是以默认的python2.6安装的，在第2步执行如下：

	pip install scipy
	
会提示这样的错误

```
	Collecting scipy
  Using cached scipy-0.16.1.tar.gz
Installing collected packages: scipy
  Running setup.py install for scipy
    Complete output from command /usr/bin/python -c "import setuptools, tokenize;__file__='/tmp/pip-build-7YhC6v/scipy/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /tmp/pip-TtkOaR-record/install-record.txt --single-version-externally-managed --compile:
    lapack_opt_info:
    openblas_lapack_info:
      libraries openblas not found in ['/usr/local/lib64', '/usr/local/lib', '/usr/lib64', '/usr/lib']
      NOT AVAILABLE

    lapack_mkl_info:
    mkl_info:
      libraries mkl,vml,guide not found in ['/usr/local/lib64', '/usr/local/lib', '/usr/lib64', '/usr/lib']
      NOT AVAILABLE

      NOT AVAILABLE

    atlas_3_10_threads_info:
    Setting PTATLAS=ATLAS
      libraries tatlas,tatlas not found in /usr/local/lib64
      libraries lapack_atlas not found in /usr/local/lib64
      libraries tatlas,tatlas not found in /usr/local/lib
      libraries lapack_atlas not found in /usr/local/lib
      libraries tatlas,tatlas not found in /usr/lib64/atlas
      libraries lapack_atlas not found in /usr/lib64/atlas
      libraries tatlas,tatlas not found in /usr/lib64/sse2
      libraries lapack_atlas not found in /usr/lib64/sse2
      libraries tatlas,tatlas not found in /usr/lib64
      libraries lapack_atlas not found in /usr/lib64
      libraries tatlas,tatlas not found in /usr/lib
      libraries lapack_atlas not found in /usr/lib
    <class 'numpy.distutils.system_info.atlas_3_10_threads_info'>
      NOT AVAILABLE

    atlas_3_10_info:
      libraries satlas,satlas not found in /usr/local/lib64
      libraries lapack_atlas not found in /usr/local/lib64
      libraries satlas,satlas not found in /usr/local/lib
      libraries lapack_atlas not found in /usr/local/lib
      libraries satlas,satlas not found in /usr/lib64/atlas
      libraries lapack_atlas not found in /usr/lib64/atlas
      libraries satlas,satlas not found in /usr/lib64/sse2
      libraries lapack_atlas not found in /usr/lib64/sse2
      libraries satlas,satlas not found in /usr/lib64
      libraries lapack_atlas not found in /usr/lib64
      libraries satlas,satlas not found in /usr/lib
      libraries lapack_atlas not found in /usr/lib
    <class 'numpy.distutils.system_info.atlas_3_10_info'>
      NOT AVAILABLE

    atlas_threads_info:
    Setting PTATLAS=ATLAS
      libraries ptf77blas,ptcblas,atlas not found in /usr/local/lib64
      libraries lapack_atlas not found in /usr/local/lib64
      libraries ptf77blas,ptcblas,atlas not found in /usr/local/lib
      libraries lapack_atlas not found in /usr/local/lib
      libraries ptf77blas,ptcblas,atlas not found in /usr/lib64/atlas
      libraries lapack_atlas not found in /usr/lib64/atlas
      libraries ptf77blas,ptcblas,atlas not found in /usr/lib64/sse2
      libraries lapack_atlas not found in /usr/lib64/sse2
      libraries ptf77blas,ptcblas,atlas not found in /usr/lib64
      libraries lapack_atlas not found in /usr/lib64
      libraries ptf77blas,ptcblas,atlas not found in /usr/lib
      libraries lapack_atlas not found in /usr/lib
    <class 'numpy.distutils.system_info.atlas_threads_info'>
      NOT AVAILABLE

    atlas_info:
      libraries f77blas,cblas,atlas not found in /usr/local/lib64
      libraries lapack_atlas not found in /usr/local/lib64
      libraries f77blas,cblas,atlas not found in /usr/local/lib
      libraries lapack_atlas not found in /usr/local/lib
      libraries f77blas,cblas,atlas not found in /usr/lib64/atlas
      libraries lapack_atlas not found in /usr/lib64/atlas
      libraries f77blas,cblas,atlas not found in /usr/lib64/sse2
      libraries lapack_atlas not found in /usr/lib64/sse2
      libraries f77blas,cblas,atlas not found in /usr/lib64
      libraries lapack_atlas not found in /usr/lib64
      libraries f77blas,cblas,atlas not found in /usr/lib
      libraries lapack_atlas not found in /usr/lib
    <class 'numpy.distutils.system_info.atlas_info'>
      NOT AVAILABLE

    /usr/lib64/python2.6/site-packages/numpy/distutils/system_info.py:1552: UserWarning:
        Atlas (http://math-atlas.sourceforge.net/) libraries not found.
        Directories to search for the libraries can be specified in the
        numpy/distutils/site.cfg file (section [atlas]) or by setting
        the ATLAS environment variable.
      warnings.warn(AtlasNotFoundError.__doc__)
    lapack_info:
      libraries lapack not found in ['/usr/local/lib64', '/usr/local/lib', '/usr/lib64', '/usr/lib']
      NOT AVAILABLE

    /usr/lib64/python2.6/site-packages/numpy/distutils/system_info.py:1563: UserWarning:
        Lapack (http://www.netlib.org/lapack/) libraries not found.
        Directories to search for the libraries can be specified in the
        numpy/distutils/site.cfg file (section [lapack]) or by setting
        the LAPACK environment variable.
      warnings.warn(LapackNotFoundError.__doc__)
    lapack_src_info:
      NOT AVAILABLE

    /usr/lib64/python2.6/site-packages/numpy/distutils/system_info.py:1566: UserWarning:
        Lapack (http://www.netlib.org/lapack/) sources not found.
        Directories to search for the sources can be specified in the
        numpy/distutils/site.cfg file (section [lapack_src]) or by setting
        the LAPACK_SRC environment variable.
      warnings.warn(LapackSrcNotFoundError.__doc__)
      NOT AVAILABLE

    Running from scipy source directory.
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "/tmp/pip-build-7YhC6v/scipy/setup.py", line 253, in <module>
        setup_package()
      File "/tmp/pip-build-7YhC6v/scipy/setup.py", line 250, in setup_package
        setup(**metadata)
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/core.py", line 135, in setup
        config = configuration()
      File "/tmp/pip-build-7YhC6v/scipy/setup.py", line 175, in configuration
        config.add_subpackage('scipy')
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/misc_util.py", line 1001, in add_subpackage
        caller_level = 2)
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/misc_util.py", line 970, in get_subpackage
        caller_level = caller_level + 1)
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/misc_util.py", line 907, in _get_configuration_from_setup_py
        config = setup_module.configuration(*args)
      File "scipy/setup.py", line 15, in configuration
        config.add_subpackage('linalg')
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/misc_util.py", line 1001, in add_subpackage
        caller_level = 2)
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/misc_util.py", line 970, in get_subpackage
        caller_level = caller_level + 1)
      File "/usr/lib64/python2.6/site-packages/numpy/distutils/misc_util.py", line 907, in _get_configuration_from_setup_py
        config = setup_module.configuration(*args)
      File "scipy/linalg/setup.py", line 20, in configuration
        raise NotFoundError('no lapack/blas resources found')
    numpy.distutils.system_info.NotFoundError: no lapack/blas resources found

    ----------------------------------------
	Command "/usr/bin/python -c "import setuptools, tokenize;__file__='/tmp/pip-build-7YhC6v/scipy/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /tmp/pip-TtkOaR-record/install-record.txt --single-version-externally-managed --compile" failed with error code 1 in /tmp/pip-build-7YhC6v/scipy
```

这是因为缺少 **lapack** 的原因，执行如下命令安装即可

	sudo yum -y install lapack-devel
	
之后再次执行，成功安装：

```
sudo pip install scipy
Collecting scipy
  Using cached scipy-0.16.1.tar.gz
Installing collected packages: scipy
  Running setup.py install for scipy
Successfully installed scipy-0.16.1
```
安装Redis的错误提示

在执行

	make test
	
提示系统需要**tcl**8.5以上版本，直接执行命令安装即可：

	yum install tcl
	
之后

	make install 
	
安装完启动 */var/log/redis/redis.log* 会有如下错误：

```
WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
```
解决方法

第一个警告两个方式解决(overcommit_memory)

1.  echo "vm.overcommit_memory=1" > /etc/sysctl.conf  或 vi /etcsysctl.conf , 然后reboot重启机器
2.  echo 1 > /proc/sys/vm/overcommit_memory  不需要启机器就生效

第二个警告解决

1. echo 511 > /proc/sys/net/core/somaxconn

overcommit_memory参数说明：

设置内存分配策略（可选，根据服务器的实际情况进行设置）
/proc/sys/vm/overcommit_memory

可选值：0、1、2。

0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。

1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。

2， 表示内核允许分配超过所有物理内存和交换空间总和的内存

注意：redis在dump数据的时候，会fork出一个子进程，理论上child进程所占用的内存和parent是一样的，比如parent占用 的内存为8G，这个时候也要同样分配8G的内存给child,如果内存无法负担，往往会造成redis服务器的down机或者IO负载过高，效率下降。所 以这里比较优化的内存分配策略应该设置为 1（表示内核允许分配所有的物理内存，而不管当前的内存状态如何）。

这里又涉及到Overcommit和OOM。

什么是Overcommit和OOM

在Unix中，当一个用户进程使用malloc()函数申请内存时，假如返回值是NULL，则这个进程知道当前没有可用内存空间，就会做相应的处理工作。许多进程会打印错误信息并退出。
Linux使用另外一种处理方式，它对大部分申请内存的请求都回复"yes"，以便能跑更多更大的程序。因为申请内存后，并不会马上使用内存。这种技术叫做Overcommit。
当内存不足时，会发生OOM killer(OOM=out-of-memory)。它会选择杀死一些进程(用户态进程，不是内核线程)，以便释放内存。

Overcommit的策略

Linux下overcommit有三种策略(Documentation/vm/overcommit-accounting)：

0. 启发式策略。合理的overcommit会被接受，不合理的overcommit会被拒绝。
1. 任何overcommit都会被接受。
2. 当系统分配的内存超过swap+N%*物理RAM(N%由vm.overcommit_ratio决定)时，会拒绝commit。
3. 
overcommit的策略通过vm.overcommit_memory设置。
overcommit的百分比由vm.overcommit_ratio设置。

\# echo 2 > /proc/sys/vm/overcommit_memory

\# echo 80 > /proc/sys/vm/overcommit_ratio

当oom-killer发生时，linux会选择杀死哪些进程
选择进程的函数是oom_badness函数(在mm/oom_kill.c中)，该函数会计算每个进程的点数(0~1000)。
点数越高，这个进程越有可能被杀死。
每个进程的点数跟oom_score_adj有关，而且oom_score_adj可以被设置(-1000最低，1000最高)。

最后启动的时候，如果执行

	sudo ./analyzer.d start

提示缺少factorial之类的库时大多是因为scipy没有装好或者版本不对应

最后测试数据时，执行：

	cd utils
	python seed_data.py

如果提示：
	
	Woops, looks like the metrics didn't make it into Horizon. Try again?
	
则需要做如下设置：

1. 找到配置文件 *settings.py* 将下面前边的 # 去掉并改为：

	HORIZON_IP = '127.0.0.1' 
	
	之后重启 ./horizon.d

2. 修改 *seed_data.py* 中的44行 **socket.gethostname()** 为 **'127.0.0.1'**

如果在其他终端浏览器中通过该机IP查看端口1500不能访问的话则修改配置文件 *settings.py* 

	WEBAPP_IP = '127.0.0.1'
	
将其值修改为对应的机器IP，并重启 ./webapp.d 即可

Graphite后端 Carbon和Whisper的安装 [点这里](http://segmentfault.com/a/1190000002533877)