title: PHPexcel单元格内换行
date: 2016-04-05 17:06:55
tags: [PHP]
---

```php
$objPHPExcel->getActiveSheet()->getStyleByColumnAndRow($k + 1, $row)->getAlignment()->setWrapText(true);

$objPHPExcel->getActiveSheet()->setCellValueByColumnAndRow($k + 1, $row, "Hello\nWorld");
```

*要换行的字符串 `Hello\nWorld` 外面必须是双引号*


