title: React-Native 学习笔记
date: 2016-05-04 19:27:12
tags: [React-Native]
toc: true
---

### 如何运行

- 先确保你已安装好了React Native 所需的依赖环境
- 在根目录下执行 npm install
- 再执行 npm start
- 最后在Xcode中点击run 运行 或者按 command + R

<!-- more -->
### 在iOS设备中运行

- 需要[苹果开发者账号](https://developer.apple.com/register)
- 打开项目中的`AppDelegate.m`文件
-  将`jsCodeLocation = [NSURL URLWithString:@"http://localhost:8081/Examples/Movies/MoviesApp.ios.bundle?platform=ios&dev=true"];`中的`localhost`修改为电脑的*IP*
- 在Xcode中选择iOS设备并点击编译和运行

*在iOS设备上运行需要在电脑中安装证书*

### 注意点

* react-native 中，图片必须明确写明大小值，不然无法显示，同时width : '100%'',这种写法不支持。
* 如果需要自适应，有几种做法：
	1. 只写高度值，不写宽度值，外层容器使用flex来做好布局，再配合resizeMode实现图片自适应即可。

		* 例子1 :
		
		 ```
<View style={{flex : 1,borderRightWidth : 1,borderRightColor: '#eeeeee'}}>
                <Image style={{height: 110,resizeMode: Image.resizeMode.contain}} source={{uri: 'http://gtms01.alicdn.com/tps/i1/TB1nif8HpXXXXc6XVXXAyLxZVXX-320-188.jpg'}} />
            </View>
		```
		
		* 例子2 :

			```
		<View style={{
  flex: 1,
  alignItems: 'stretch',
}}>
  <Image ssource={{uri: 'http://gtms01.alicdn.com/tps/i1/TB1nif8HpXXXXc6XVXXAyLxZVXX-320-188.jpg'}} style={{ flex: 1 }} />
</View>
```

	2 . 使用Dimensions来获取设备viewport的宽高
	
	```
	var Dimensions = require('Dimensions');
var { width, height } = Dimensions.get('window');
var image = (
  <Image style={{width: width, height: 100 }} source={{uri: 'http://gtms01.alicdn.com/tps/i1/TB1nif8HpXXXXc6XVXXAyLxZVXX-320-188.jpg'}} />
);
```