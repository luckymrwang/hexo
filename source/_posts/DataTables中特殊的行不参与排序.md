title: DataTables中特殊的行不参与排序
date: 2015-08-24 14:52:00
tags: [DataTables]
---

DataTables中可以直接设置某列不参与排序，但是对于特殊的行其工具本身没有设置项 *不过对于table底部的行可以添加<tfoot></tfoot>标签来固定，并且不参与排序*

下面将介绍在<tbody></tbody>中最上面的特殊行不参与排序的方法，分一行和多行的情况
<!-- more -->
### 只有一行时：

- 在<tbody>中<tr>添加class类 如：`<tr class="no-sort">` *这是不参与排序的行*

- 在 `<script type="text/script">` 中的代码如下：

~~~javascript
<script type="text/javascript">
	jQuery(function($) {
		var $tr = $('.no-sort');
		var mySpecialRow = $tr.html();
		$tr.remove();
		var table = $('#<?php echo $table_id;?>').dataTable({
			"fnDrawCallback": function( oSettings ) {
				$('#example tbody').prepend(mySpecialRow);
			}
		});
});
</script>
~~~

### 多行时：

- 在<tbody>中<tr>添加class类 如：`<tr class="no-sort">` *这是不参与排序的行*

- 在 `<script type="text/script">` 中的代码如下：

~~~javascript
<script type="text/javascript">
	jQuery(function($) {
		var mySpecialRow;		$tr.each(function(){			mySpecialRow = mySpecialRow+"<tr>"+$(this).html()+"</tr>";			$(this).remove();
		}
		var table = $('#<?php echo $table_id;?>').dataTable({
			"fnDrawCallback": function( oSettings ) {
				$('#example tbody').prepend(mySpecialRow);
			}
		});
});
</script>
~~~

### 效果如下：

之前：
![no-sort](/images/no-sort_foot.png)

之后：
![no-sort](/images/no-sort_body.png)



