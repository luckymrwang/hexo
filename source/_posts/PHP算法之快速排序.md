title: PHP算法之快速排序
date: 2015-08-21 15:43:30
tags: [PHP]
---
### 思路分析

选择一个基准元素，通常选择第一个元素或者最后一个元素。通过一趟扫描，将待排序列分成两部分，一部分比基准元素小，一部分大于等于基准元素。此时基准元素在其排好序后的正确位置，然后再用同样的方法递归地排序划分的两部分。
<!-- more -->

    $arr = array(12, 242, 3, 20, 11,50);
    function kuaisu($arr){
        $len = count($arr);
        if($len <= 1){
            return $arr;
        }
        $left = $right = array();
        //选择基准值
        $mark = $arr[0];
        //依次循环比较
        for($i=1; $i<$len; $i++){
            //把小值放入左边，把大值放入右边
            ($mark > $arr[$i]) ? ($left[] = $arr[$i]) : ($right[] = $arr[$i]);
        }
        //对$left/$right进行递归循环比较
        $left = kuaisu($left);
        $right = kuaisu($right);
        //合并
        return array_merge($left, array($mark), $right);
    }
    echo implode(',', kuaisu($arr));