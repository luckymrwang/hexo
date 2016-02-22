title: RESTful API版本控制策略
date: 2015-12-30 15:23:15
tags: [API]
toc: true
---
### 前言
做RESTful开放平台，一方面其API变动越少， 对API调用者越有利；另一方面，没有人可以预测未来，系统在发展的过程中，不可避免的需要添加新的资源，或者修改现有资源。因此，改动升级必不可少，但是，作为平台开发者，你必须有觉悟：一旦你的API开放出去，有人开始用了，你就不能只管自己Happy了，你对平台的任何改动都需要考虑对当前用户的影响。因此，做开放平台，你从第一个API的设计就需要开始API的版本控制策略问题，API的版本控制策略就像是开放平台和平台用户之间的长期协议，其设计的好坏将直接决定用户是否使用该平台，或者说用户在使用之后是否会因为某次版本升级直接弃用该平台。 

<!-- more -->
### 版本控制策略模式 
API的版本控制策略通常有3种模式：

第一种：**The Knot**：无版本，即平台的API永远只有一个版本，所有的用户都必须使用最新的API，任何API的修改都会影响到平台所有的用户。甚至平台的整个生态系统。 

![the-knot](/images/the-knot.png)

第二种：**Point-to-Point**：点对点，即平台的API版本自带版本号，用户根据自己的需求选择使用对应的API，需要使用新的API特性，用户必须自己升级。 

![point-to-point](/images/point-to-point.png)

第三种：**Compatible Versioning**：兼容性版本控制，和The Knot一样，平台只有一个版本，但是最新版本需要兼容以前版本的API行为。 

![compatible-versioning](/images/compatible-versioning.png)

三种策略对整个平台在升级API的开销对比如下：

![api](/images/api.png)

以上信息来源于[这里](http://www.ebpml.org/blog2/index.php/2013/11/25/understanding-the-costs-of-versioning)

针对上面的论述，首先，API一定得有版本，否则升级对于用户来说将是噩梦。其次，要做到Compatible Versioning有现实的限制，毕竟API升级时，不可避免的会出现无法兼容老版本的状况，因此，版本控制需要结合第二种和第三种模式，即提供一个统一的兼容版本入口，同时对于不能兼容历史版本的API保留历史版本，支持用户能够调用到历史版本的API。另外，对历史版本的API支持一定要有时间和用户限制，即老版API支持到一定时间就删除，新用户必须使用新版API，否则一个API有10个版本会让平台的维护非常痛苦。 

### URI vs Request Parameter vs Media Type

在RESTful API领域，关于如何做版本控制，目前业界比较主流的有3种做法： 

第一种：URI， 即在URI中直接标记使用的是哪个版本，无版本号URI默认使用最新版本。如下： 

```javascript
http://server:port/api/customers/1234
http://server:port/api/v3.0/customers/1234
```

好处：

-	直接可以在URI中直观的看到API版本
-	可以直接在浏览器的查看各个版本API的结果

坏处：


- 版本号在URI中破坏了REST的HATEOAS（hypermedia as the engine of application state）规则
- 版本号和资源之间并无直接关系

第二种：Request Parameter， 即在每个请求后添加一个version参数，表示请求的是哪个版本。如下：

```javascript
http://server:port/api/customer/123?version=2
```

这种做法其实就是URI方式的变种，好坏处也都一样。

第三种： Mdedia Type， 即在HTTP请求的header中使用Media Type标记该请求想获取的资源， 同样的可以不设置或设置通用的Media Type表示最新版本的API。

```javascript
===>
GET /customer/123 HTTP/1.1
Accept: application/vnd.xianlinbox.customer-v3+json

<===
HTTP/1.1 200 OK
Content-Type: application/vnd.xianlinbox.customer-v3+json

{"customer":
	{"name":"Xianlinbox"}
}
```

好处：

- 遵循了REST的设计风格， 

坏处：

- 版本不直观，需要能设置header的client才能调用查看该API的效果。
