title: PHP算法之选择排序
date: 2015-08-21 15:53:25
tags: [PHP]
---
### 思路分析

在要排序的一组数中，选出最小的一个数与第一个位置的数交换。然后在剩下的数当中再找最小的与第二个位置的数交换，如此循环到倒数第二个数和最后一个数比较为止。

<!-- more -->

```php
    $arr = array(12, 242, 3, 20, 11,50);
    function xuanzhi($arr){
        $len = count($arr);
        for($i=0; $i<$len; $i++){
            //假设最小的位置
            $p = $i;
            //循环比较
            for($j=$i+1; $j<$len; $j++){
                if($arr[$p] > $arr[$j]){
                    $p = $j;    //交换最小值位置
                }
            }
            //经过上面循环比较后，已经确定了最小值位置
            //判断最小值是否和假设一致
            //不一致，则交换位置
            if($p != $i){
                $tmp = $arr[$p];
                $arr[$p] = $arr[$i];
                $arr[$i] = $tmp;
            }
        }
        return $arr;
    }
    echo implode(',', xuanzhi($arr));
```