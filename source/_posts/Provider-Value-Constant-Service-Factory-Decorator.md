title: 'Provider,Value,Constant,Service,Factory,Decorator'
date: 2016-01-30 16:30:13
tags: [AngularJS]
---

`provider`, `value`, `constant`, `service`, `factory` 都是 `provider`。(后面解释原因）

#### Provider

Provider可以为应用提供通用的服务，形式可以是常量，也可以是对象。

<!-- more -->
比如我们在controller里常用的 $http 就是AngularJS框架提供的provider

```js
myApp.controller(‘MainController', function($scope, $http) {
    $http.get(…)
}
```

定制一个Provider：

```js
//定义:
$provide.provider('age', {
    start: 10,
    $get: function() {
      return this.start + 2;
    }
});
//或
$provide.provider('age', function($filterProvider){
    this.start = 10;
    this.$get = function() {
      return this.start + 2;
    };
});

//调用:
app.controller('MainCtrl', function($scope, age) {
  $scope.age = age; //12
});
```

provider的基本原则就是通过实现$get方法来在应用中注入单例，使用的时候拿到的age就是$get执行后的结果。 上面例子中的两种定义方法都可以～

#### Factory

如果只想定义一个$get,则需要引入factory：

```js
$provide.provider('myDate', {
    $get: function() {
      return new Date();
    }
});
//可以写成
$provide.factory('myDate', function(){
    return new Date();
});

//调用:
app.controller('MainCtrl', function($scope, myDate) {
  $scope.myDate = myDate; //current date
});
```

直接第二个参数就是$get要对应的函数实现，代码简单了很多有没有？！

#### Service

如果不仅想定义一个$get, 里面还就返回个new出来的已有js类，则需要引入service：

```js
$provide.provider('myDate', {
    $get: function() {
      return new Date();
    }
});
//可以写成
$provide.factory('myDate', function(){
    return new Date();
});
//可以写成
$provide.service('myDate', Date);
```

#### Value VS Constant

如果只想定义一个$get,而且就返回个常量：

```js
$provide.value('pageCount', 7);
$provide.constant('pageCount', 7);
```

##### 区别一：value可以被修改，constant一旦声明无法被修改

```js
$provide.decorator('pageCount', function($delegate) {
    return $delegate + 1;
});
```

decorator可以用来修改（修饰）已定义的provider们，除了constant～

##### 区别二：value不可在config里注入，constant可以

```js
myApp.config(function(pageCount){
    //可以得到constant定义的'pageCount'
});
```

#### 通过底层实现代码看关系～

```js
function provider(name, provider_) {
    if (isFunction(provider_)) {
        provider_ = providerInjector.instantiate(provider_);
    }
    if (!provider_.$get) {
        throw Error('Provider ' + name + ' must define $get factory method.');
    }
    return providerCache[name + providerSuffix] = provider_;
}

function factory(name, factoryFn) { return provider(name, { $get: factoryFn }); }

function service(name, constructor) {
    return factory(name, ['$injector', function($injector) {
        return $injector.instantiate(constructor);
    }]);
}

function value(name, value) { return factory(name, valueFn(value)); }

function constant(name, value) {
    providerCache[name] = value;
    instanceCache[name] = value;
}

function decorator(serviceName, decorFn) {
    var origProvider = providerInjector.get(serviceName + providerSuffix),
        orig$get = origProvider.$get;

    origProvider.$get = function() {
        var origInstance = instanceInjector.invoke(orig$get, origProvider);
        return instanceInjector.invoke(decorFn, null, {$delegate: origInstance});
    };
}
```

本文引自[这里](http://hellobug.github.io/blog/angularjs-providers/)
