title: "Rolling cURL:PHP并发最佳实践"
date: 2015-05-20 16:15:41
tags: [PHP]
---
前言：在实际项目或者自己编写小工具(比如新闻聚合,商品价格监控,比价)的过程中, 通常需要从第3方网站或者API接口获取数据, 在需要处理1个URL队列时, 为了提高性能, 可以采用cURL提供的curl_multi_*族函数实现简单的并发.

本文将探讨两种具体的实现方法, 并对不同的方法做简单的性能对比.
<!-- more -->
## 经典cURL并发机制及其存在的问题
经典的cURL实现机制在网上很容易找到, 比如参考[PHP在线手册](http://php.net/manual/en/function.curl-multi-init.php)在线手册，不过这里我附上我们BI项目中使用的多并发方式:

	public static function multi_curl_new($requests) {
		$mh = curl_multi_init();
		$chs = array();
		foreach ($requests as $key => $req) {
			$timeout = empty($req['timeout']) ? 20 : $req['timeout'];
			$ch = curl_init($req['url']);
			curl_setopt($ch, CURLOPT_TIMEOUT, $timeout);
			curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
			curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
			if(defined('CURLOPT_IPRESOLVE') && defined('CURL_IPRESOLVE_V4')) {
				curl_setopt($ch, CURLOPT_IPRESOLVE, CURL_IPRESOLVE_V4);
			}
			if(empty($req['not_need_proxy'])) {
				MY_Pull::set_curl_proxy($req['big_app_id'], $ch);
			}
			curl_multi_add_handle($mh, $ch);
			$chs[$key] = $ch;
		}
		
		$active = null;
		do {
			$mrc = curl_multi_exec($mh, $active);
		} while($mrc == CURLM_CALL_MULTI_PERFORM);
		
		while($active && $mrc==CURLM_OK) {
			if(curl_multi_select($mh) != -1) {
				do {
					$mrc = curl_multi_exec($mh, $active);
				} while($mrc == CURLM_CALL_MULTI_PERFORM);
			}
			usleep(100);
		}
		
		$rets = array();
		foreach($chs as $key => $ch) {
			$rets[$key] = curl_multi_getcontent($ch);
			curl_multi_remove_handle($mh, $ch);
			curl_close($ch);
		}
		curl_multi_close($mh);
		
		return $rets;
	}

首先将所有的URL压入并发队列, 然后执行并发过程, 等待所有请求接收完之后进行数据的解析等后续处理. 在实际的处理过程中, 受网络传输的影响, 部分URL的内容会优先于其他URL返回, 但是经典cURL并发必须等待最慢的那个URL返回之后才开始处理, 等待也就意味着CPU的空闲和浪费. 如果URL队列很短, 这种空闲和浪费还处在可接受的范围, 但如果队列很长, 这种等待和浪费将变得不可接受.

##改进的Rolling cURL并发方式

仔细分析不难发现经典cURL并发还存在优化的空间, 优化的方式时当某个URL请求完毕之后尽可能快的去处理它, 边处理边等待其他的URL返回, 而不是等待那个最慢的接口返回之后才开始处理等工作, 从而避免CPU的空闲和浪费. 闲话不多说, 下面贴上具体的实现:

	function rolling_curl($urls, $delay) {
		$queue = curl_multi_init();
		$map = array();
	
		foreach ($urls as $url) {
			$ch = curl_init();
	
			curl_setopt($ch, CURLOPT_URL, $url);
			curl_setopt($ch, CURLOPT_TIMEOUT, 1);
			curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
			curl_setopt($ch, CURLOPT_HEADER, 0);
			curl_setopt($ch, CURLOPT_NOSIGNAL, true);
	
			curl_multi_add_handle($queue, $ch);
			$map[(string) $ch] = $url;
		}
	
		$responses = array();
		do {
			while (($code = curl_multi_exec($queue, $active)) == CURLM_CALL_MULTI_PERFORM) ;
	
			if ($code != CURLM_OK) { break; }
	
			// a request was just completed -- find out which one
			while ($done = curl_multi_info_read($queue)) {
	
				// get the info and content returned on the request
				$info = curl_getinfo($done['handle']);
				$error = curl_error($done['handle']);
				$results = callback(curl_multi_getcontent($done['handle']), $delay);
				$responses[$map[(string) $done['handle']]] = compact('info', 'error', 'results');
	
				// remove the curl handle that just completed
				curl_multi_remove_handle($queue, $done['handle']);
				curl_close($done['handle']);
			}
	
			// Block for data in / output; error handling is done by curl_multi_exec
			if ($active > 0) {
				curl_multi_select($queue, 0.5);
			}
	
		} while ($active);
	
		curl_multi_close($queue);
		return $responses;
	}

*引自文章 http://www.searchtb.com/2012/06/rolling-curl-best-practices.html*