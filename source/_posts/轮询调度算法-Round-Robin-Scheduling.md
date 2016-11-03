title: '轮询调度算法(Round-Robin Scheduling)'
date: 2016-11-03 10:45:05
tags: [Go]
---

　　毫无疑问，随着互联网、移动网络接入成本的降低，互联网正在日益深入地走入我们的生活，越来越成为人们获取信息的高效平台，ICP行业也顺势呈现出强劲的成长趋势，成为互联网迅猛发展形势下最大的受益者，也直接促成了从web1.0到web2.0以及社区、博客、视频等一系列互联网时代的更迭和运营模式的变动。

　　但是随着各站点访问量和信息交流量的迅猛增长，如何使用最小的资源成本，提高网络的效率，最优化用户体验，已经成为网络管理人员不得不面对的挑战。

　　从技术上讲，就是ICP行业面临的网络资源有效利用问题，也就是如何进行对网络的访问分流，以便能够快速响应用户反应，即：负载均衡。

<!-- more -->
　　从这篇文章起，我们将讲述在负载均衡技术实现中的核心技术：负载均衡算法(算法)的原理及其实现，使大家对负载均衡底层技术有一个深刻的了解。这些算法是负载均衡设备中的核心实现基础。

　　本篇文章先讲述轮询调度算法 (Round-Robin)及其在此基础上改进型的权重轮询算法 (Weighted Round-Robin)。
　　
### 轮询调度算法(Round-Robin Scheduling)

　　轮询调度算法的原理是每一次把来自用户的请求轮流分配给内部中的服务器，从1开始，直到N(内部服务器个数)，然后重新开始循环。

　　算法的优点是其简洁性，它无需记录当前所有连接的状态，所以它是一种无状态调度。

　　轮询调度算法流程

　　假设有一组服务器N台，S = {S1, S2, …, Sn}，一个指示变量i表示上一次选择的服务器ID。变量i被初始化为N-1。其算法如下：

```
j = i;

do

{

	j = (j + 1) mod n;

	i = j;

	return Si;

} while (j != i);

return NULL;
```

　　轮询调度算法假设所有服务器的处理性能都相同，不关心每台服务器的当前连接数和响应速度。当请求服务间隔时间变化比较大时，轮询调度算法容易导致服务器间的负载不平衡。

　　所以此种均衡算法适合于服务器组中的所有服务器都有相同的软硬件配置并且平均服务请求相对均衡的情况。
　　
### 权重轮询调度算法(Weighted Round-Robin Scheduling)

　　上面所讲的轮询调度算法并没有考虑每台服务器的处理能力，在实际情况中，可能并不是这种情况。由于每台服务器的配置、安装的业务应用等不同，其处理能力会不一样。所以，我们根据服务器的不同处理能力，给每个服务器分配不同的权值，使其能够接受相应权值数的服务请求。

　　权重轮询调度算法流程

　　假设有一组服务器S = {S0, S1, …, Sn-1}，W(Si)表示服务器Si的权值，一个指示变量i表示上一次选择的服务器，指示变量cw表示当前调度的权值，max(S)表示集合S中所有服务器的最大权值，gcd(S)表示集合S中所有服务器权值的最大公约数。变量i初始化为-1，cw初始化为零。其算法如下：

```
while (true) {

  i = (i + 1) mod n;

  if (i == 0) {

     cw = cw - gcd(S);

     if (cw <= 0) {

       cw = max(S);

       if (cw == 0)

         return NULL;

     }

  }

  if (W(Si) >= cw)

    return Si;

}
```

　　由于权重轮询调度算法考虑到了不同服务器的处理能力，所以这种均衡算法能确保高性能的服务器得到更多的使用率，避免低性能的服务器负载过重。所以，在实际应用中比较常见。

### 总结

　　轮询调度算法以及权重轮询调度算法的特点是实现起来比较简洁，并且实用。目前几乎所有的负载均衡设备均提供这种功能。

本文引自[这里](http://blog.163.com/s_u/blog/static/1330836720105233102894/)

### golang实现权重轮询调度算法(Weighted Round-Robin Scheduling)

```go
package main
     
    import (
    	"fmt"
    	"time"
    )
     
    var slaveDns = map[int]map[string]interface{}{
    	0: {"connectstring": "root@tcp(172.16.0.164:3306)/shiqu_tools?charset=utf8", "weight": 2},
    	1: {"connectstring": "root@tcp(172.16.0.165:3306)/shiqu_tools?charset=utf8", "weight": 4},
    	2: {"connectstring": "root@tcp(172.16.0.166:3306)/shiqu_tools?charset=utf8", "weight": 8},
    }
     
    var i int = -1 //表示上一次选择的服务器
    var cw int = 0 //表示当前调度的权值
    var gcd int = 2 //当前所有权重的最大公约数 比如 2，4，8 的最大公约数为：2
     
    func getDns() string {
    	for {
    		i = (i + 1) % len(slaveDns)
    		if i == 0 {
    			cw = cw - gcd
    			if cw <= 0 {
    				cw = getMaxWeight()
    				if cw == 0 {
    					return ""
    				}
    			}
    		}
     
    		if weight, _ := slaveDns[i]["weight"].(int); weight >= cw {
    			return slaveDns[i]["connectstring"].(string)
    		}
    	}
    }
     
    func getMaxWeight() int {
    	max := 0
    	for _, v := range slaveDns {
    		if weight, _ := v["weight"].(int); weight >= max {
    			max = weight
    		}
    	}
     
    	return max
    }
     
    func main() {
     
    	note := map[string]int{}
     
    	s_time := time.Now().Unix()
     
    	for i := 0; i < 100; i++ {
    		s := getDns()
    		fmt.Println(s)
    		if note[s] != 0 {
    			note[s]++
    		} else {
    			note[s] = 1
    		}
    	}
     
    	e_time := time.Now().Unix()
     
    	fmt.Println("total time: ", e_time-s_time)
     
    	fmt.Println("--------------------------------------------------")
     
    	for k, v := range note {
    		fmt.Println(k, " ", v)
    	}
    }
```

![结果](/images/round-robin-scheduling.png)

本文引自[这里](http://studygolang.com/articles/8940)



