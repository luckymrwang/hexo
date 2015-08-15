title: "Apache的Order All,Deny详解"
date: 2015-06-03 11:46:40
tags: [Apache]
---

Allow和Deny可以用于apache的conf文件或者.htaccess文件中（配合Directory, Location, Files等），用来控制目录和文件的访问授权。

##Tips

- 修改完配置后要保存好并重启Apache服务，配置才能生效

- 开头字母不分大小写

- Allow、Deny语句不分先后顺序，谁先谁后不影响最终判断结果；但都会被判断到
<!-- more -->
- Order语句中，“Allow,Deny”之间“有且只有”一个逗号（英文格式的），而且先后顺序很重要

- Apache有一条缺省规则，“Order Allow,Deny”本身就默认了拒绝所有的意思，因为Deny在Allow的后面；同理，“Order Deny,Allow”本身默认的是允许所有；当然，最终判断结果还要综合下面的Allow、Deny语句中各自所包含的范围；（也就是说Order语句后面可以没有Allow、Deny语句）

- Allow、Deny语句中，第二个单词一定是“from”，否则Apache会因错而无法启动，

- “Order Allow,Deny”代表先判断Allow语句再判断Deny语句，反之亦然。

##判断规则

- 首先判断默认的

- 然后判断逗号前的

- 最后判断逗号后的

- 最终按顺序叠加而得出判断结果

##举例说明

*配置文件中的“#”号后边的数字表示配置项起作用的先后顺序。*

	<Directory "D:/TRS/Apache2.2.17/cgi-bin">
    	Order allow,deny  #1
    	Allow from all #2
    	Deny from 192.9.200.69 #3
	</Directory>

**1.规律**

当我们看到一个apache的配置时，可以从下面的角度来理解。一默认，二顺序，三重叠。

**2.上面配置说明**

- 一默认

Order allow,deny ，这句话的作用是配置allow和deny的顺序，默认只有最后一个关键字起作用，这里起作用的关键字就是“deny”，默认拒绝所有请求。为了便于理解，我们可以画一个圆，圆的背景色涂上黑色，我们给这个圆起个编号，叫圆1。

- 二顺序

由于上边的Order指出判断的顺序是先判断Allow的规则，然后才是Deny的规则。所以我们要先判断Allow的请求，由于该请求中配置的是Allow from all，
所以表示该请求允许所有请求。这时我们再画一个圆，背景色涂上白色，我们给圆起个编号，叫圆2。

我们再来看Deny的判断规则，由于 Deny from 192.9.200.69 ，表示拒绝来自ip地址为“192.9.200.69”，所以我们可以画出一块红色区域，表示“192.9.200.69”，我们把这块区域叫区域3。

*注意:即使把“Allow from all”写在“Deny from 192.9.200.69”下面，依然是需要先判断Allow规则，也就是说只有Order才能决定Allow和Order的优先顺序。*

- 三重叠

我们把上边产生的圆1、圆2和区域3依次从下往上堆叠在一起。每个层都是不透明的，这时我们可以看到最终效果是除了“192.9.200.69”这块红色区域外，其他的所有都是白色区域。也就是只有“192.9.200.69”这个ip地址没有权限访问该目录，其他的请求都有权限访问该目录。

##例子

*配置文件中的“#”号后边的数字表示配置项起作用的先后顺序。*

- 只允许192.9.200.69请求访问目录

		<Directory "D:/TRS/Apache2.2.17/cgi-bin">
			Order deny,allow #1.默认允许全部请求
			Allow from 192.9.200.69 #3.重叠，允许IP192.9.200.69的请求
			Deny from all #2.按照顺序，先判断deny规则，拒绝所有请求
		</Directory>

- 允许所有请求访问目录

		<Directory "D:/TRS/Apache2.2.17/cgi-bin">
			Order deny,allow #1.默认允许全部请求
			Allow from all #3.重叠，允许所有请求
			Deny from 192.9.200.69 #2.按照顺序，先判断deny规则，拒绝192.9.200.69的请求
		</Directory>

- 拒绝所有请求访问目录

		<Directory "D:/TRS/Apache2.2.17/cgi-bin">
			Order allow,deny #1.默认拒绝全部请求
			Allow from 192.9.200.69 #2.顺序，允许 192.9.200.69请求
			Deny from  all#3.重叠，拒绝所有请求
		</Directory>

- 除了192.9.200.69的请求外，其他请求都可以访问目录

		<Directory "D:/TRS/Apache2.2.17/cgi-bin">
			Order allow,deny #1.默认拒绝全部请求
			Deny from  192.9.200.69#3.重叠，拒绝192.9.200.69请求
			Allow from all #2.顺序，允许所有请求
		</Directory>
