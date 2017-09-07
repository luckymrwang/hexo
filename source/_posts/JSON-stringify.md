title: JSON.stringify()
date: 2015-12-24 20:36:05
tags: [JS]
toc: true
---

### 概述

**JSON.stringify()** 方法可以将任意的 JavaScript 值序列化成 JSON 字符串。

### 语法

	JSON.stringify(value[, replacer [, space]])
	
### 参数

#### value
将要序列化成 JSON 字符串的值。

#### replacer  `可选`
如果该参数是一个函数，则在序列化过程中，被序列化的值的每个属性都会经过该函数的转换和处理；如果该参数是一个数组，则只有包含在这个数组中的属性名才会被序列化到最终的 JSON 字符串中。关于该参数更详细的解释和示例，请参考使用原生的 JSON 对象一文。

#### space `可选`
指定缩进用的空白字符串，用于美化输出（pretty-print）。

### 描述

关于序列化，有下面五点注意事项：

- 非数组对象的属性不能保证以特定的顺序出现在序列化后的字符串中。
- 布尔值、数字、字符串的包装对象在序列化过程中会自动转换成对应的原始值。
- undefined、任意的函数以及 symbol 值，在序列化过程中会被忽略（出现在非数组对象的属性值中时）或者被转换成 null（出现在数组中时）。
- 所有以 symbol 为属性键的属性都会被完全忽略掉，即便 replacer 参数中强制指定包含了它们。
- 不可枚举的属性会被忽略

```js
JSON.stringify({});                        // '{}'
JSON.stringify(true);                      // 'true'
JSON.stringify("foo");                     // '"foo"'
JSON.stringify([1, "false", false]);       // '[1,"false",false]'
JSON.stringify({ x: 5 });                  // '{"x":5}'

JSON.stringify({x: 5, y: 6});              
// '{"x":5,"y":6}' 或者 '{"y":6,"x":5}' 都可能
JSON.stringify([new Number(1), new String("false"), new Boolean(false)]); 
// '[1,"false",false]'
JSON.stringify({x: undefined, y: Object, z: Symbol("")}); 
// '{}'
JSON.stringify([undefined, Object, Symbol("")]);          
// '[null,null,null]' 
JSON.stringify({[Symbol("foo")]: "foo"});                 
// '{}'
JSON.stringify({[Symbol.for("foo")]: "foo"}, [Symbol.for("foo")]);
// '{}'
JSON.stringify({[Symbol.for("foo")]: "foo"}, function (k, v) {
  if (typeof k === "symbol"){
    return "a symbol";
  }
});

// '{}'  

// 不可枚举的属性默认会被忽略：
JSON.stringify( Object.create(null, { x: { value: 'x', enumerable: false }, y: { value: 'y', enumerable: true } }) );
// '{"y":"y"}'
```

### space 参数

space 参数用来控制结果字符串里面的间距。如果是一个数字, 则在字符串化时每一级别会比上一级别缩进多这个数字值的空格（最多10个空格）；如果是一个字符串，则每一级别会比上一级别多缩进用该字符串（或该字符串的前十个字符）。

	JSON.stringify({ a: 2 }, null, " ");   // '{\n "a": 2\n}'

使用制表符（\t）来缩进：

```js
JSON.stringify({ uno: 1, dos : 2 }, null, '\t')
// '{            \
//     "uno": 1, \
//     "dos": 2  \
// }'
```

### toJSON 方法

如果一个被序列化的对象拥有 toJSON 方法，那么该 toJSON 方法就会覆盖该对象默认的序列化行为：不是那个对象被序列化，而是调用 toJSON 方法后的返回值会被序列化，例如：

```js
var obj = {
  foo: 'foo',
  toJSON: function () {
    return 'bar';
  }
};
JSON.stringify(obj);      // '"bar"'
JSON.stringify({x: obj}); // '{"x":"bar"}'
```

### 使用 JSON.stringify 结合 localStorage 的例子

一些时候，你想存储用户创建的一个对象，并且，即使在浏览器被关闭后仍能恢复该对象。下面的例子是 JSON.stringify 适用于这种情形的一个样板：

```js
// 创建一个示例数据
var session = {
    'screens' : [],
    'state' : true
};
session.screens.push({"name":"screenA", "width":450, "height":250});
session.screens.push({"name":"screenB", "width":650, "height":350});
session.screens.push({"name":"screenC", "width":750, "height":120});
session.screens.push({"name":"screenD", "width":250, "height":60});
session.screens.push({"name":"screenE", "width":390, "height":120});
session.screens.push({"name":"screenF", "width":1240, "height":650});

// 使用 JSON.stringify 转换为 JSON 字符串
// 然后使用 localStorage 保存在 session 名称里
localStorage.setItem('session', JSON.stringify(session));

// 然后是如何转换通过 JSON.stringify 生成的字符串，该字符串以 JSON 格式保存在 localStorage 里
var restoredSession = JSON.parse(localStorage.getItem('session'));

// 现在 restoredSession 包含了保存在 localStorage 里的对象
console.log(restoredSession);
```

### 相关链接
- [使用原生得JSON对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Using_native_JSON)