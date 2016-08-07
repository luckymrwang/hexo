title: beego中带参数的UrlFor和urlfor的用法讲解
date: 2016-07-31 14:12:52
tags: [beego,go]
toc: true
---

我们知道，代码里面可以使用`UrlFor`或者模板里面使用`urlfor`来根据自定义`Controller`和方法来生成`url`。这里模板中使用的`urlfor`其实是`UrlFor`注册的模板方法，二者功能完全相同，一个用于代码中，另外一个用于模板中。

步骤如下：

- 自定义`Controller`，并且实现对应的方法`Method`
- 使用`beego.Router`注册路由，将自定义`Controller`实例和方法`Method`联系在一起
- 使用`UrlFor`函数在代码中或者`urlfor`在模板中生成`url`

上面的是我们经常会使用的方法，这里再分享一个带参数的`UrlFor`或者`urlfor`的用法。

<!-- more -->
```go
package main

import (
	"fmt"
	"github.com/astaxie/beego"
)

type UserController struct {
	beego.Controller
}

func (this UserController) List() {

}

func main() {
	userController := UserController{}

	//注册路由
	beego.Router("/user/list/:name/:age", &userController, "*:List")
	beego.Router("/user/list", &userController, "*:List")

	//创建url
	//{{urlfor "UserController.List" ":name" "astaxie" ":age" "25"}}
	url := userController.UrlFor("UserController.List", ":name", "astaxie", ":age", "25")
	//输出 /user/list/astaxie/25
	fmt.Println(url)

	//{{urlfor "UserController.List" "name" "astaxie" "age" "25"}}
	url = userController.UrlFor("UserController.List", "name", "astaxie", "age", "25")
	//输出 /user/list?name=astaxie&age=25
	fmt.Println(url) 
	
	// 也可以直接这样写,不带参数不接收返回值
	beego.UrlFor("UserController.List")
	
}
```

我们上面分别使用了路由参数的方式和表单参数的方式来分别创建了两个`url`。对应的注释中是它们在模板中的使用方法。

```go
/*
/user/list/astaxie/25
*/
{{urlfor "UserController.List" ":name" "astaxie" ":age" "25"}}
/*
/user/list?name=astaxie&age=25
*/
{{urlfor "UserController.List" "name" "astaxie" "age" "25"}}

/*
/user/list
*/
{{urlfor "UserController.List"}}
```

需要注意的是，在模板中给方法提供的参数中间是用空格隔开的，另外注意路由参数和表单参数两种方式下，路由的注册方式和参数名称的提供方式。带冒号`(:)`的参数名是路由参数。另外参数的提供方式是依次以`key-value`方式提供的。

本文引自[这里](http://golanghome.com/post/364)