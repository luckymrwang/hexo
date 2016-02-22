title: PHPExcel隐藏Sheet
date: 2015-12-08 17:27:27
tags: [PHPExcel]
---

PHPExcel隐藏Sheet语句：

```php
$objPHPExcel->createSheet();

//在07版以上不起作用，在excel中取消隐藏后即可显示
$objPHPExcel->getSheetByName('Worksheet')->setSheetState(PHPExcel_Worksheet::SHEETSTATE_HIDDEN);

//适用于07版及以上，在excel中不能取消隐藏，可在visual_basic中查看
$objPHPExcel->getSheetByName('Worksheet')->setSheetState(PHPExcel_Worksheet::SHEETSTATE_VERYHIDDEN);
```