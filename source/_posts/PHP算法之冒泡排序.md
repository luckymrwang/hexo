title: PHP算法之冒泡排序
date: 2015-08-21 15:40:12
tags: [PHP]
---
### 思路分析

在要排序的一组数中，对当前还未排好的序列，从前往后对相邻的两个数依次进行比较和调整，让较大的数往下沉，较小的往上冒。即，每当两相邻的数比较后发现它们的排序与排序要求相反时，就将它们互换。
<!-- more -->

```php
    $arr = array(12, 242, 3, 20, 11,50);
    function maopao($arr){
       $len = count($arr);
        for($i=0;$i<$len;$i++){
            for($j=$i+1;$j<$len;$j++){
                if($arr[$i]>$arr[$j]){
                    $tmp = $arr[$i];
                    $arr[$i] = $arr[$j];
                    $arr[$j] = $tmp;
                }
            }
        }
        return $arr;
    }
    echo implode(',', maopao($arr));
```