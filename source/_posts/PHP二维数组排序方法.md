title: PHP二维数组排序方法
date: 2016-09-21 14:35:17
tags: [PHP]
---

例如像下面的数组：

```php
$users = array(
    array('name' => 'tom', 'age' => 20),
    array('name' => 'anny', 'age' => 18)，
    array('name' => 'jack', 'age' => 22)
);
```

#### 使用array_multisort

先将age提取出来存储到一维数组里，然后按照age升序排列。具体代码如下：

```php
$ages = array();
foreach ($users as $user) {
    $ages[] = $user['age'];
}
array_multisort($ages, SORT_ASC, $users);
```

执行后，$users就是排序好的数组了，可以打印出来看看。如果需要先按年龄升序排列，再按照名称升序排列，方法同上，就是多提取一个名称数组出来，最后的排序方法这样调用：

```php
array_multisort($ages, SORT_ASC, $names, SORT_ASC, $users);
```

#### 使用usort

使用这个方法最大的好处就是可以自定义一些比较复杂的排序方法。例如按照名称的长度降序排列：

```php
usort($users, function($a, $b) {
      $al = strlen($a['name']);
      $bl = strlen($b['name']);
      if ($al == $bl)
        return 0;
      return ($al > $bl) ? -1 : 1;
});
```

这里使用了匿名函数，如果有需要也可以单独提取出来。其中$a, $b可以理解为$users数组下的元素，可以直接索引name值，并计算长度，而后比较长度就可以了。