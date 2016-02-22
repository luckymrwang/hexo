title: Draggable拖拽克隆后可拖拽
date: 2015-12-11 15:26:23
tags: [jQuery]
---

使用jQuery UI 拖动控件克隆后继续能拖动，代码如下：

``` html
<div class="form-group" style="float:left;">
	<input type="text" id="form-field-1" placeholder="input" class="col-xs-10 col-sm-5">
</div>

<div class="widget-main" style="height:800px;"></div>

```

``` javascript
$(".form-group").draggable({
		appendTo: ".widget-main",
		helper: "clone",
		revert: "invalid",
		cursor: "move"
	});
    $( ".widget-main" ).droppable({
      activeClass: "bg",
      hoverClass: "",
      accept: ".form-group",
      drop: function(event, ui) {
		// $(this).append($(ui.draggable).clone()); 
       $(ui.helper).clone().appendTo($(this)).draggable({appendTo: ".widget-main",revert: "invalid"}); //继续有拖动功能
		$(ui.helper).remove(); //去掉克隆功能
      }
    }).sortable({
      items: "li:not(.placeholder)",
      sort: function() {
        // 获取由 droppable 与 sortable 交互而加入的条目
        // 使用 connectWithSortable 可以解决这个问题，但不允许您自定义 active/hoverClass 选项
        $(this).removeClass("ui-state-default");
      }
    });
```