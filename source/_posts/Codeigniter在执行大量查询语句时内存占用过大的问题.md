title: "Codeigniter在执行大量查询语句时内存占用过大的问题"
date: 2015-05-28 15:18:51
tags: [CI]
---

在跑审计前250名用户清单数据时，总是因为内存占用过大而异常退出。查看代码，也就是两个循环嵌套连接MySQL查询数据然后处理显示出来（不会存在一次取出数据超内存的情况），具体循环代码如下：<!-- more -->

	for($i = 0; $i < $total_count['total_count']; $i += $count){
		echo "mem".memory_get_usage()."\n";
		$items = $this->Model_data_table_payurl->get_all_order_by_user_id($big_app_id, $first_second, $last_second, $v['user_id'], $i, $count);
		echo "mem".memory_get_usage()."\n";
		foreach($items as $key => $row) {
			$audit_cfg = MY_Audit::get_audit_sub_app_cfg($big_app_id, $row['trade_type']);
			MY_Time::set_timezone($big_app_id);
			$date = date("Y-m-d H:i:s",$row['createtime']);
			$ips = $this->get_ips_by_day($big_app_id, $row['zoneid'], $date, $row['userid']);	
			echo $game_desc."\t".$audit_cfg['partner']."\t".$app_ret['appname']."\t".$audit_cfg['desc']."\t".$audit_cfg['sales_mode']."\t".$audit_cfg['country']."\t".$audit_cfg['shore']."\t".
				$date."\t".$row['curType']."\t".$row['userid']."\t".$row['zoneid']."\t".$row['name']."\t".$row['num']."\t".$row['cost']."\t".$row['ip']."\t".$ips."\n";
			unset($items[$key]);
		}
	}

其中`echo "mem".memory_get_usage()."\n";`显示查询数据库之前与之后的内存占用情况，发现内存是一直递增的，信息显示如下：

	mem5933840
	mem9584368
	mem10304976
	mem12652816
	mem12918072
	mem14033464
	mem14451576
	mem16399352

调查及解决的过程：

- 因为每次几乎都是以*2M*的基数递增，所以考虑先是不连接数据库，将`$items`赋值一固定的*2M*数据，执行后发现内存维持在一个几乎固定的数值内，不会继续递增，所以怀疑就是连接数据库时资源没有释放，一直累积

- 在查看CI手册时*生成查询记录集*中对*$query->free_result()*的解释：

该函数将会释放当前查询所占用的内存并删除其关联的资源标识。通常来说，PHP 将会脚本执行结束后自动释放内存。如果当前执行的请求将要花很长时间并且占用比较大的资源时，该函数可以在一定程度上降低资源的消耗：

	$query = $this->db->query('SELECT title FROM my_table');
	foreach ($query->result() as $row){
   		echo $row->title;
	}
	$query->free_result(); // $query 将不再可用

	$query2 = $this->db->query('SELECT name FROM some_table');
	$row = $query2->row();
	echo $row->name;
	$query2->free_result(); // $query2 将不再可用

于是利用此方法来释放资源，但最终发现每次调用完只释放了几K的内存，犹如杯水车薪并不能解决内存持续递增的问题

- 在打印*$db*对象时（如何打印[看这里](http://luckymrwang.github.io/2015/05/20/CI%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B1%BB/)）发现其中一项*[save_queries] => 1*，所以打开*system/database/DB_driver.php*查看CI源码，看到：


	var $save_queries	= TRUE;

	// Save the  query for debugging
	if ($this->save_queries == TRUE)
	{
		$this->queries[] = $sql;
	}

	// Start the Query Timer
	$time_start = list($sm, $ss) = explode(' ', microtime());

	// Run the Query
	if (FALSE === ($this->result_id = $this->simple_query($sql)))
	{
		if ($this->save_queries == TRUE)
		{
			$this->query_times[] = 0;
		}

		// This will trigger a rollback if transactions are being used
		$this->_trans_status = FALSE;

		if ($this->db_debug)
		{
			...

- CI db会将所有的查询sql和和sql执行时间保存下来，为了debug。

所以到这里问题已很明显，我就将*$save_queries*的值改为*FALSE*，然后重新执行，结果如下：

	mem5924328
	mem9573632
	mem5971680
	mem8307064
	mem5971504
	mem7091896
	mem5971528
	mem7916000
之前的内存递增已解决。

当你执行大数据时，记得`$save_queries = FALSE;`
或在model中加入 $this->db->save_queries = FALSE;