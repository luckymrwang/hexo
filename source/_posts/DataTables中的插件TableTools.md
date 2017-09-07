title: DataTables中的插件TableTools
date: 2015-08-15 17:36:56
tags: [DataTables]
---

TableTools是一个DataTables中的插件，它能够将table表中的数据拷贝到粘贴板，导出到excel甚至是pdf格式。

[下载](https://www.datatables.net/download/packages)以及[使用方法](https://www.datatables.net/extensions/tabletools/)：*官网给出的初始化如下*

<!-- more -->

```js
	//Example initialisation
	$(document).ready( function () {
    	$('#example').dataTable( {
    	    "dom": 'T<"clear">lfrtip',
    	    "tableTools": {
    	        "sSwfPath": "/swf/copy_csv_xls_pdf.swf"
    	    }
    	} );
	} );
```

### 出现的问题及解决办法：

1.

- 描述：引入js文件和css文件后，相应的Button会出现在table的右上角，但是只有`Print`按钮起作用
- 原因：`sSwfPath`后面的`swf`文件没有引用到
- 解决办法*这是用PHP输出的全路径*：

```php
"sSwfPath": "<?php echo base_url(); ?>assets/swf/copy_csv_xls_pdf.swf"
```

2.

- 导出excel时中文乱码的
- 解决办法：在引用tabletools时加入 `charset="gbk"` 这个属性,如下：
	
	<script type="text/javascript" src="<?php echo base_url(); ?>assets/js/dataTables.tableTools.min.js" charset="gbk"></script>

3.

- 自定义按钮以及文字描述

```js
	"tableTools": {
		"sSwfPath": "<?php echo base_url(); ?>assets/swf/copy_csv_xls_pdf.swf",
		"aButtons":[
			{"sExtends":"copy", "sButtonText":"拷贝到粘贴板"},
			{"sExtends":"xls", "sButtonText":"导出Excel"},
			{"sExtends":"print", "sButtonText":"打印预览"}
		]
	}
```