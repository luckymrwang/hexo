title: PHP连接Access
date: 2016-05-31 22:09:26
tags: [PHP]
toc: true
---

## PHP连接Access数据库

工作中需要中控的联网考勤打卡机将考勤数据同步到公司的OA系统中，但中控的打卡机只能通过网络将数据发送到其自己的考勤系统中，而其考勤系统用的数据库是Microsoft Access，所以才有了这个需求。

### 环境

- 必须是Windows系统
- LAMP或LNMP必须是32位版的，否则报错连接不上

### PHP需要的扩展
<!-- more -->
通过运行`phpinfo()`查看，应该包含以下扩展，否则修改`php.ini`文件

![](/images/access_03.png)

### SQL命令

#### 连接Access

```php
<?php
$dbName = $_SERVER["DOCUMENT_ROOT"] . "products\products.mdb";
if (!file_exists($dbName)) {
    die("Could not find database file.");
}
$db = new PDO("odbc:DRIVER={Microsoft Access Driver (*.mdb)}; DBQ=$dbName; Uid=; Pwd=;");
```

#### SELECT

```php
<?php
$sql  = "SELECT price FROM product";
$sql .= " WHERE id = " . $productId;

$result = $db->query($sql);
$row = $result->fetch();

$productPrice = $row["price"];
```

```php
<?php
$sql  = "SELECT p.name, p.description, p.price";
$sql .= "  FROM product p, product_category pc";
$sql .= " WHERE p.id  = pc.productId";
$sql .= "  AND pc.category_id = " . $categoryId;
$sql .= " ORDER BY name";

$result = $db->query($sql);
while ($row = $result->fetch()) {
    $productName        = $row["name"];
    $productDescription = $row["description"];
    $productPrice       = $row["price"];
}
```
#### UPDATE

```php
<?php
$sql  = "UPDATE product";
$sql .= "   SET description = " . $db->quote($strDescription) . ",";
$sql .= "       price       =  " . $strPrice . ",";
$sql .= "       sale_status = " . $db->quote($strDescription);
$sql .= " WHERE id = " . $productId;

$db->query($sql);
```

#### INSERT

```php
<?php
$sql  = "INSERT INTO product";
$sql .= "       (name, description, price, sale_status) ";
$sql .= "VALUES (" . $db->quote($strName) . ", " . $db->quote($strDescription) . ", " . $strPrice . ", " . $db->quote($strStatus) . ")";

$db->query($sql);
```

#### DELETE

```php
<?php
$sql  = "DELETE";
$sql .= "  FROM product";
$sql .= " WHERE id = " . $productId;

$db->query($sql);
```




