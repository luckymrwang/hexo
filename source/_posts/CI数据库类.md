title: "CI数据库类"
date: 2015-05-20 15:51:51
tags: [CI]
---
今天在查看CI的用户指南数据库类时，提示连接多个数据库时要用

	$DB1 = $this->load->database('group_one', TRUE);
	$DB2 = $this->load->database('group_two', TRUE);

通过设置函数的第二个参数为TRUE(boolean)来返回一个数据库对象。
<!-- more -->
*当你使用这种方法，你将用对象名来执行操作命令而不是用户向导模式，也就是说，你将用以下方式执行数据库操作：*

	$DB1->query();
	$DB1->result();
	etc...

而不是：

	$this->db->query();
	$this->db->result();
	etc...


*译注：要连接多个数据库请先设置 config/database.php 中的 $db['xxxxxx']['pconnect'] = FALSE; 这是 mysql_pconnect() 造成的问题，和 CI 无关。*

我借用BI项目测试后返回一个对象，具体内容是这样的：

	CI_DB_mysql_driver Object
        (
            [dbdriver] => mysql
            [_escape_char] => `
            [_like_escape_str] => 
            [_like_escape_chr] => 
            [delete_hack] => 1
            [_count_string] => SELECT COUNT(*) AS 
            [_random_keyword] =>  RAND()
            [use_set_names] => 
            [ar_select] => Array
                (
                )

            [ar_distinct] => 
            [ar_from] => Array
                (
                )

            [ar_join] => Array
                (
                )

            [ar_where] => Array
                (
                )

            [ar_like] => Array
                (
                )

            [ar_groupby] => Array
                (
                )

            [ar_having] => Array
                (
                )

            [ar_keys] => Array
                (
                )

            [ar_limit] => 
            [ar_offset] => 
            [ar_order] => 
            [ar_orderby] => Array
                (
                )

            [ar_set] => Array
                (
                )

            [ar_wherein] => Array
                (
                )

            [ar_aliased_tables] => Array
                (
                )

            [ar_store_array] => Array
                (
                )

            [ar_caching] => 
            [ar_cache_exists] => Array
                (
                )

            [ar_cache_select] => Array
                (
                )

            [ar_cache_from] => Array
                (
                )

            [ar_cache_join] => Array
                (
                )

            [ar_cache_where] => Array
                (
                )

            [ar_cache_like] => Array
                (
                )

            [ar_cache_groupby] => Array
                (
                )

            [ar_cache_having] => Array
                (
                )

            [ar_cache_orderby] => Array
                (
                )

            [ar_cache_set] => Array
                (
                )

            [ar_no_escape] => Array
                (
                )

            [ar_cache_no_escape] => Array
                (
                )

            [username] => root
            [password] => 
            [hostname] => localhost
            [database] => ga
            [dbprefix] => 
            [char_set] => utf8
            [dbcollat] => utf8_general_ci
            [autoinit] => 1
            [swap_pre] => 
            [port] => 
            [pconnect] => 
            [conn_id] => Resource id #112
            [result_id] => Resource id #113
            [db_debug] => 1
            [benchmark] => 0.00244402885437
            [query_count] => 1
            [bind_marker] => ?
            [save_queries] => 1
            [queries] => Array
                (
                    [0] => SELECT *
						FROM (`apps`)
						WHERE `appid` =  1009
                )

            [query_times] => Array
                (
                    [0] => 0.00244402885437
                )

            [data_cache] => Array
                (
                )

            [trans_enabled] => 1
            [trans_strict] => 1
            [_trans_depth] => 0
            [_trans_status] => 1
            [cache_on] => 
            [cachedir] => 
            [cache_autodel] => 
            [CACHE] => 
            [_protect_identifiers] => 1
            [_reserved_identifiers] => Array
                (
                    [0] => *
                )

            [stmt_id] => 
            [curs_id] => 
            [limit_used] => 
            [stricton] => 
        )

变量非常多，例如 `[queries]` 也会记录着查询过的语句，`[query_time]` 记录着相应语句查询所需要的时间等等。