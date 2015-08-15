title: "BOSS页面"
date: 2015-05-18 16:04:45
tags: [BI项目]
---

##月用户新增进度
*功能：展示各个游戏，各个区域（国内_Anndroid,国内_IOS,国外_Android&IOS）的开服量、前日新增用户数、昨日新增用户数、上月同期新增用户数、本月当前新增用户数、上月总新增用户数、预计本月新增用户数*

具体流程如下：

1. 权限判断【超级管理员、运营权限管理员、BOSS、运营经理】可以直接查看

2. 遍历 *appconfig.php* 文件通过 *shore* 来分成 *境内* 和 *境外* 两部分，之后调用 `_get_shore_item` 函数来整合不同区域的数据。*在该函数中做如下处理*：根据手机平台类型（Android、IOS）来区分统计，最终返回数据形式如下：<!-- more -->

		Array
		(
    		[android] => Array
			(
            	[servers_count] => 607
            	[before_yesterday_new_users] => 89708
            	[yesterday_new_users] => 50479
            	[last_month_now_new_users] => 1089132
            	[this_month_now_new_users] => 1026796
            	[last_month_total_new_users] => 1645542
            	[this_month_expected_new_users] => 1768371
        	)

    		[ios] => Array
        	(
            	[servers_count] => 366
            	[before_yesterday_new_users] => 13441
            	[yesterday_new_users] => 11216
            	[last_month_now_new_users] => 235718
            	[this_month_now_new_users] => 219045
            	[last_month_total_new_users] => 364892
            	[this_month_expected_new_users] => 377244
        	)

    		[total_servers_count] => 660
		)

3. 汇总不同平台的数据
4. 汇总每个游戏的数据【 *开服数量需要单独算* 】
5. 所有游戏总计

##月收入进度总览
（与 *月用户新增进度* 相同）