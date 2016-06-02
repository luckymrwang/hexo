title: React-Native
date: 2016-05-24 11:10:17
tags: [React-Native]
toc: true
---

### 简介

React Native使你能够在Javascript和React的基础上获得完全一致的开发体验，构建世界一流的原生APP。

React Native着力于提高多平台开发的开发效率 —— 仅需学习一次，编写任何平台。(Learn once, write anywhere)

Facebook已经在多项产品中使用了React Native，并且将持续地投入建设React Native。

国内应用场景代表：iPad天猫客户端

<!-- more -->
### 需要知识

 - html,css
 - ES6
 - react.js
 - flex (布局)
 
### 环境需求

- Node.js 4.0或更高版本

* iOS开发环境：安装Xcode 7.0或者更高版本

* Android开发环境：安装Android SDK

### 准备工作

在所有依赖的软件都已经安装完毕后，只需要输入两条命令就可以创建一个React Native工程。

	npm install -g react-native-cli

react-native-cli是一个终端命令，它可以完成其余的设置工作。它可以通过npm安装。刚才这条命令会往你的终端安装一个叫做react-native的命令。这个安装过程你只需要进行一次。

	react-native init FirstProject

这个命令会初始化一个工程、下载React Native的所有源代码和依赖包，最后在FirstProject/iOS/FirstProject.xcodeproj和FirstProject/android/app下分别创建一个新的XCode工程和一个gradle工程。

### 开发

- 开发iOS版本，可以在XCode中打开刚刚创建的工程 `(AwesomePrjoect/iOS/FirstProject.xcodeproj)`，然后只要按下`⌘+R`就可以构建并运行。这个操作会同时打开一个用于实现动态代码加载的Node服务`（React Packager）`。所以当修改代码，只需要在模拟器中按下`⌘+R`，而无需重新在XCode中编译。

- 开发Android版本，先连接设备或启动模拟器，然后在 `FirstProject`目录下运行`react-native run-android`，就会构建工程并自动安装到模拟器或者设备，同时启动用于实现动态代码加载的Node服务。当修改代码之后，只需要打开摇一摇菜单(摇一下设备，或者按下设备的`Menu`键，或者在模拟器上按下`F2`或`Page Up`，`Genymotion`按下`⌘+M`)，然后在菜单中点击`“Reload JS”`。

在本向导中我会创建一个简单的Movies应用，它可以获取25个上映中的电影，然后把它们在一个ListView中显示。

### 模拟数据

注：React Native从0.18之后，新建项目默认已经采用了ES6语法，[ES6与ES5的区别](http://bbs.reactnative.cn/topic/15/react-react-native-%E7%9A%84es5-es6%E5%86%99%E6%B3%95%E5%AF%B9%E7%85%A7%E8%A1%A8)，ES6语法[阮一峰的书](http://es6.ruanyifeng.com/)。

在真正从Rotten Tomatoes(一个国外的电影社区)抓取数据之前，先制造一些模拟数据来练练手。可以把下面代码放在index.ios.js和index.android.js中的import语句之后或者任意位置：

```js
var MOCKED_MOVIES_DATA = [
  {title: '标题', year: '2015', posters: {thumbnail: 'http://i.imgur.com/UePbdph.jpg'}},
];
```
### 展现一个电影

接下来要展现一个电影，绘制它的标题、年份、以及缩略图。渲染缩略图需要用到Image组件，所以把Image添加到对React的import列表中。

```js
import React, { Component } from 'react';
import { 
  AppRegistry,
  Image,
  StyleSheet,
  Text,
  View,
} from 'react-native';
```

然后修改一下render函数，这样就可以把上面创建的模拟数据渲染出来。

```js
render() {
    var movie = MOCKED_MOVIES_DATA[0];
    return (
      <View style={styles.container}>
        <Text>{movie.title}</Text>
        <Text>{movie.year}</Text>
        <Image source={{uri: movie.posters.thumbnail}} />
      </View>
    );
  }
```

按下`⌘+R`或者`Reload JS`，现在你应该能看到文字`"Title"`和`"2015"`，但现在Image组件没渲染任何东西，这是因为还没有为图片指定想要渲染的宽和高。这通过样式来实现。当修改样式的时候，也应该清理掉不再使用的样式。

```js
var styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  thumbnail: {
    width: 53,
    height: 81,
  },
});
```

然后把它应用到Image组件上：

```js
<Image
	source={{uri: movie.posters.thumbnail}}
	style={styles.thumbnail}
/>
```

按下`⌘+R`或者`Reload JS`，现在图片应该可以被渲染出来了。

### 添加样式

```js
+---------------------------------+
|+-------++----------------------+|
||       ||        标题          ||
|| 图片  ||                      ||
||       ||        年份          ||
|+-------++----------------------+|
+---------------------------------+
```

所以需要增加一个container来实现一个水平布局内嵌套一个垂直布局。

```js
return (
  <View style={styles.container}>
    <Image
      source={{uri: movie.posters.thumbnail}}
      style={styles.thumbnail}
    />
    <View style={styles.rightContainer}>
      <Text style={styles.title}>{movie.title}</Text>
      <Text style={styles.year}>{movie.year}</Text>
    </View>
  </View>
);
```

和之前相比并没有太多变化，增加了一个container来包装文字，然后把它移到了Image的后面（因为它们最终在图片的右边）。样式修改：

```js
container: {
    flex: 1,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
},
```

*`flexDirection: 'row'` 让主容器的成员从左到右横向布局，而非默认的从上到下纵向布局。* [flex布局](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html?utm_source=tuicool)

在style对象里增加另一个样式：

```js
rightContainer: {
    flex: 1,
  },
```

*让rightContainer在父容器中占据Image之外剩下的全部空间*

给文字添加样式:

```js
  title: {
    fontSize: 20,
    marginBottom: 8,
    textAlign: 'center',
  },
  year: {
    textAlign: 'center',
  },
```

再⌘+R或者Reload JS查看效果。

### 拉取真正的数据

*用一个样例来代替从Rotten Tomatoes的API拉取数据，这个样例数据放在React Native的Github库中。*

把下面的常量放到文件的最开头（通常在require下面）来创建请求数据所需的地址常量REQUEST_URL

```js
var REQUEST_URL = 'https://raw.githubusercontent.com/facebook/react-native/master/docs/MoviesExample.json';
```

首先在应用中创建一个初始的null状态，这样可以通过this.state.movies == null来判断数据是不是已经被抓取到了。

在服务器响应返回的时候执行this.setState({movies: moviesData})来改变这个状态。把下面这段代码放到React类的render函数之前：

```js
  constructor(props) {
    super(props);
    this.state = {
      movies: null,  //这里放自定义的state变量及初始值
    };
  }
```

组件加载完毕之后，就可以向服务器请求数据。componentDidMount是React组件的一个生命周期方法，它会在组件刚加载完成的时候调用一次，以后不会再被调用。

```js
  componentDidMount() {
    this.fetchData();
  }
```

为组件添加fetchData函数。在Promise调用链结束后执行this.setState({movies:data})。在React的工作机制下，setState实际上会触发一次重新渲染的流程，此时render函数被触发，发现this.state.movies不再是null。在Promise调用链的最后调用了done() —— 这样可以抛出异常而不是简单忽略。

```js
  fetchData() {
    fetch(REQUEST_URL)
      .then((response) => response.json())
      .then((responseData) => {
        this.setState({
          movies: responseData.movies,
        });
      })
      .done();
  }
```

修改render函数。在电影数据加载完毕之前，先渲染一个“加载中”的视图；而如果电影数据已经加载完毕了，则渲染第一个电影数据。

```js
 render() {
    if (!this.state.movies) {
      return this.renderLoadingView();
    }

    var movie = this.state.movies[0];
    return this.renderMovie(movie);
  }

  renderLoadingView() {
    return (
      <View style={styles.container}>
        <Text>
          正在加载电影数据……
        </Text>
      </View>
    );
  }

  renderMovie(movie) {
    return (
      <View style={styles.container}>
        <Image
          source={{uri: movie.posters.thumbnail}}
          style={styles.thumbnail}
        />
        <View style={styles.rightContainer}>
          <Text style={styles.title}>{movie.title}</Text>
          <Text style={styles.year}>{movie.year}</Text>
        </View>
      </View>
    );
  }
```

⌘+R或者Reload JS，首先看到“正在加载电影数据……”，然后在响应数据到达之后，看到第一个电影的信息。

### ListView

让应用能够渲染所有的数据而不是仅仅第一部电影。用到的就是ListView组件。

*为什么建议把内容放到ListView里？比起直接渲染出所有的元素，或是放到一个ScrollView里有什么优势？这是因为尽管React很高效，渲染一个可能很大的元素列表还是会很慢。ListView会安排视图的渲染，只显示当前在屏幕上的那些元素。而那些已经渲染好了但移动到了屏幕之外的元素，则会从原生视图结构中移除（以提高性能）。*

首先要做的事情：在文件最开头，从React中引入ListView。

```js
import React, { Component } from 'react';
import React, {
  AppRegistry,
  Image,
  ListView,
  StyleSheet,
  Text,
  View
} from 'react-native';
```
修改render函数。当有数据之后，渲染一个包含多个电影信息的ListView，而不仅仅是单个的电影。

```js
  render() {
    if (!this.state.loaded) {
      return this.renderLoadingView();
    }

    return (
      <ListView
        dataSource={this.state.dataSource}
        renderRow={this.renderMovie}
        style={styles.listView}/>
    );
  }
```

dataSource接口用来在ListView的整个更新过程中判断哪些数据行发生了变化。

在constructor生成的初始状态中添加一个空白的dataSource。另外，现在要把数据存储在dataSource中，所以不再另外用this.state.movies来保存数据。可以在state里用一个布尔型的属性(this.state.loaded)来判断数据加载是否已经完成了。

```js
  constructor(props) {
    super(props);
    this.state = {
      dataSource: new ListView.DataSource({
        rowHasChanged: (row1, row2) => row1 !== row2,
      }),
      loaded: false,
    };
  }
```

同时也要修改fetchData方法来把数据更新到dataSource里：

```js
  fetchData() {
    fetch(REQUEST_URL)
      .then((response) => response.json())
      .then((responseData) => {
        this.setState({
          dataSource: this.state.dataSource.cloneWithRows(responseData.movies),
          loaded: true,
        });
      })
      .done();
  }
```

最后，再在styles对象里给ListView添加一些样式。

```js
  listView: {
    paddingTop: 20,
    backgroundColor: '#F5FCFF',
  },
```

⌘+R或者Reload JS查看最终效果。

添加导航器，搜索，加载更多，等等。

### 最终的代码

```js
/**
 * Sample React Native App
 * https://github.com/facebook/react-native
 */

import React, {
  Component,
} from 'react';

import {
  AppRegistry,
  Image,
  ListView,
  StyleSheet,
  Text,
  View,
} from 'react-native';

var API_KEY = '7waqfqbprs7pajbz28mqf6vz';
var API_URL = 'http://api.rottentomatoes.com/api/public/v1.0/lists/movies/in_theaters.json';
var PAGE_SIZE = 25;
var PARAMS = '?apikey=' + API_KEY + '&page_limit=' + PAGE_SIZE;
var REQUEST_URL = API_URL + PARAMS;

class FirstProject extends Component {
  constructor(props) {
    super(props);
    this.state = {
      dataSource: new ListView.DataSource({
        rowHasChanged: (row1, row2) => row1 !== row2,
      }),
      loaded: false,
    };
  }

  componentDidMount() {
    this.fetchData();
  }

  fetchData() {
    fetch(REQUEST_URL)
      .then((response) => response.json())
      .then((responseData) => {
        this.setState({
          dataSource: this.state.dataSource.cloneWithRows(responseData.movies),
          loaded: true,
        });
      })
      .done();
  }

  render() {
    if (!this.state.loaded) {
      return this.renderLoadingView();
    }

    return (
      <ListView
        dataSource={this.state.dataSource}
        renderRow={this.renderMovie}
        style={styles.listView}
      />
    );
  }

  renderLoadingView() {
    return (
      <View style={styles.container}>
        <Text>
          Loading movies...
        </Text>
      </View>
    );
  }

  renderMovie(movie) {
    return (
      <View style={styles.container}>
        <Image
          source={{uri: movie.posters.thumbnail}}
          style={styles.thumbnail}
        />
        <View style={styles.rightContainer}>
          <Text style={styles.title}>{movie.title}</Text>
          <Text style={styles.year}>{movie.year}</Text>
        </View>
      </View>
    );
  }
};

var styles = StyleSheet.create({
  container: {
    flex: 1,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  rightContainer: {
    flex: 1,
  },
  title: {
    fontSize: 20,
    marginBottom: 8,
    textAlign: 'center',
  },
  year: {
    textAlign: 'center',
  },
  thumbnail: {
    width: 53,
    height: 81,
  },
  listView: {
    paddingTop: 20,
    backgroundColor: '#F5FCFF',
  },
});

AppRegistry.registerComponent('FirstProject', () => FirstProject);
```
