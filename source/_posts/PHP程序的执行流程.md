title: "PHP程序的执行流程"
date: 2015-05-27 14:37:15
tags: [PHP]
---
*为了以后能开发PHP扩展，就一定要了解PHP的执行顺序。这篇文章就是为C开发PHP扩展做铺垫。*

前言：Web环境我们假设为Apache。在编译PHP的时候,为了能够让Apache支持PHP，我们会生成一个mod_php5.so的模块。Apache加载这个模块，在url访问.php文件的时候，就会转给mod_php5.so模块来处理。这个就是我们常说的SAPI。英文名字是：Server Application Programming Interface。SAPI其实是一个统称，其下有 ISAPI，CLI SAPI，CGI等。有了它，就可以很容易的跟其他东西交互，比如APACHE,IIS,CGI等。
<!-- more -->
Apache启动后会将mod_pho5.so模块的hook handler注册进来，当Apache检测到访问的url是一个php文件时，这时候就会把控制权交给SAPI。进入到SAPI后，首先会执行sapi/apache/mod_php5.c 文件的php_init_handler函数，这里摘录一段代码：

	static void php_init_handler(server_rec *s, pool *p)
	{
    	register_cleanup(p, NULL, (void (*)(void *))apache_php_module_shutdown_wrapper, (void (*)(void *))php_module_shutdown_for_exec);
    	if (!apache_php_initialized) {
    	    apache_php_initialized = 1;
    	    #ifdef ZTS
    	    tsrm_startup(1, 1, 0, NULL);
    	    #endif
    	    sapi_startup(&amp;apache_sapi_module);
    	    php_apache_startup(&amp;apache_sapi_module);
    	}
    	#if MODULE_MAGIC_NUMBER &gt;= 19980527
    	{
    	    TSRMLS_FETCH();
    	    if (PG(expose_php)) {
    	        ap_add_version_component("PHP/" PHP_VERSION);
    	    }
    	}
    	#endif
	}

该函数主要调用两个函数：sapi_startup(&apache_sapi_module); php_apache_startup(&apache_sapi_module);

	SAPI_API void sapi_startup(sapi_module_struct *sf)
	{
    	sf-&gt;ini_entries = NULL;
    	sapi_module = *sf;
    	.................
    	sapi_globals_ctor(&amp;sapi_globals);
    	................
 	
    	virtual_cwd_startup(); /* Could use shutdown to free the main cwd but it would just slow it down for CGI */
 
    	..................
 
    	reentrancy_startup();
	}

sapi_startup创建一个 sapi_globals_struct结构体。sapi_globals_struct保存了Apache请求的基本信息，如服务器信息，Header，编码等。sapi_startup执行完毕后再执行php_apache_startup。

	static int php_apache_startup(sapi_module_struct *sapi_module)
	{
    	if (php_module_startup(sapi_module, &amp;apache_module_entry, 1) == FAILURE) {
        return FAILURE;
    	} else {
        	return SUCCESS;
    	}
	}

php_module_startup 内容太多，这里介绍一下大致的作用：

1. 初始化zend_utility_functions 结构.这个结构是设置zend的函数指针,比如错误处理函数,输出函数,流操作函数等
2. 设置环境变量
3. 加载php.ini配置
4. 加载php内置扩展
5. 写日志
6. 注册php内部函数集
7. 调用 php_ini_register_extensions,加载所有外部扩展
8. 开启所有扩展
9. 一些清理操作

***加载php.ini配置***

	if (php_init_config(TSRMLS_C) == FAILURE) {
    	return FAILURE;
	}

php_init_config函数会在这里检查所有php.ini配置，并且找到所有加载的模块，添加到php_extension_lists结构中。

***加载php内置扩展***

调用 zend_register_standard_ini_entries加载所有php的内置扩展，如array,mysql等。

***调用 php_ini_register_extensions,加载所有外部扩展***

main/php_ini.c

	void php_ini_register_extensions(TSRMLS_D)
	{
    	zend_llist_apply(&amp;extension_lists.engine, php_load_zend_extension_cb TSRMLS_CC);
    	zend_llist_apply(&amp;extension_lists.functions, php_load_php_extension_cb TSRMLS_CC);
 
    	zend_llist_destroy(&amp;extension_lists.engine);
    	zend_llist_destroy(&amp;extension_lists.functions);
	}

zend_llist_apply函数遍历extension_lists 执行会掉函数 php_load_php_extension_cb

php_load_php_extension_cb

	static void php_load_zend_extension_cb(void *arg TSRMLS_DC)
	{
    	zend_load_extension(*((char **) arg));
	}

该函数最后调用

	if ((module_entry = zend_register_module_ex(module_entry TSRMLS_CC)) == NULL) {
    	DL_UNLOAD(handle);
    	return FAILURE;
	}

将扩展信息放到 Hash表module_registry中，Zend/zend_API.c

	if (zend_hash_add(&amp;module_registry, lcname, name_len+1, (void *)module, sizeof(zend_module_entry), (void**)&amp;module_ptr)==FAILURE) {
    	zend_error(E_CORE_WARNING, "Module '%s' already loaded", module-&gt;name);
    	efree(lcname);
    	return NULL;
	}

最后，zend_startup_modules(TSRMLS_C); 对模块进行排序，并检测是否注册到module_registry HASH表里。zend_startup_extensions(); 执行extension->startup(extension);启动扩展。
