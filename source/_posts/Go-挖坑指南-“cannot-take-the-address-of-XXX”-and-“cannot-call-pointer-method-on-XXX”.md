title: >-
  Go 挖坑指南: “cannot take the address of XXX” and “cannot call pointer method on
  XXX”
date: 2020-09-28 08:29:13
tags: [Go]
---

### 示例

<!-- more -->
```go
package main

type B struct {
    Id int
}

func New() B {
    return B{}
}

func New2() *B {
    return &B{}
}

func (b *B) Hello() {
    return
}

func (b B) World() {
    return
}

func main() {
    // 方法的接收器为 *T 类型
    New().Hello() // 编译不通过

    b1 := New()
    b1.Hello() // 编译通过

    b2 := B{}
    b2.Hello() // 编译通过

    (B{}).Hello() // 编译不通过
    B{}.Hello()   // 编译不通过

    New2().Hello() // 编译通过

    b3 := New2()
    b3.Hello() // 编译通过

    b4 := &B{} // 编译通过
    b4.Hello() // 编译通过

    (&B{}).Hello() // 编译通过

    // 方法的接收器为 T 类型
    New().World() // 编译通过

    b5 := New()
    b5.World() // 编译通过

    b6 := B{}
    b6.World() // 编译通过

    (B{}).World() // 编译通过
    B{}.World()   // 编译通过

    New2().World() // 编译通过

    b7 := New2()
    b7.World() // 编译通过

    b8 := &B{} // 编译通过
    b8.World() // 编译通过

    (&B{}).World() // 编译通过
}
```

### 输出结果

```go
./main.go:25:10: cannot call pointer method on New()
./main.go:25:10: cannot take the address of New()
./main.go:33:10: cannot call pointer method on B literal
./main.go:33:10: cannot take the address of B literal
./main.go:34:8: cannot call pointer method on B literal
./main.go:34:8: cannot take the address of B literal
```

### 问题总结

>假设 T 类型的方法上接收器既有 T 类型的，又有 *T 指针类型的，那么就不可以在不能寻址的 T 值上调用 *T 接收器的方法

- &B{} 是指针，可寻址
- B{} 是值，不可寻址
- b := B{} b是变量，可寻址

### 延伸思考

Go 语言规范中规定了可寻址(addressable)对象的定义：

>For an operand x of type T, the address operation &x generates a pointer of type *T to x. The operand must be addressable, that is, either a variable, pointer indirection, or slice indexing operation; or a field selector of an addressable struct operand; or an array indexing operation of an addressable array. As an exception to the addressability requirement, x may also be a (possibly parenthesized) composite literal. If the evaluation of x would cause a run-time panic, then the evaluation of &x does too.

>对于类型为 T 的操作数 x，地址操作符 &x 将生成一个类型为 *T 的指针指向 x。操作数必须可寻址，即，变量、间接指针、切片索引操作，或可寻址结构体的字段选择器，或可寻址数组的数组索引操作。作为可寻址性要求的例外，x 也可为（圆括号括起来的）复合字面量。如果对 x 的求值会引起运行时恐慌，那么对 &x 的求值也会引起恐慌。

>For an operand x of pointer type *T, the pointer indirection *x denotes the variable of type T pointed to by x. If x is nil, an attempt to evaluate *x will cause a run-time panic.

>对于指针类型为 *T 的操作数 x，间接指针 *x 表示类型为 T 的值指向 x。若 x 为 nil，尝试求值 *x 将会引发运行时恐慌。
>

以下几种是可寻址的：

- 一个变量: &x
- 指针引用(pointer indirection): &*x
- slice 索引操作(不管 slice 是否可寻址): &s[1]
- 可寻址 struct 的字段: &point.X
- 可寻址数组的索引操作: &a[0]
- composite literal 类型: &struct{ X int }{1}

下列情况 x 是不可以寻址的，不能使用 &x 取得指针：

- 字符串中的字节
- map 对象中的元素
- 接口对象的动态值(通过 type assertions 获得)
- 常数
- literal 值(非 composite literal)
- package 级别的函数
- 方法 method(用作函数值)
- 中间值(intermediate value):
	- 函数调用
	- 显式类型转换
	- 各种类型的操作（除了指针引用 pointer dereference 操作 *x):
		- channel receive operations
		- sub-string operations
		- sub-slice operations
		-	加减乘除等运算符

有几个点需要解释下：

- 常数为什么不可以寻址?

>如果可以寻址的话，我们可以通过指针修改常数的值，破坏了常数的定义。

- map 的元素为什么不可以寻址？

>两个原因，如果对象不存在，则返回零值，零值是不可变对象，所以不能寻址，如果对象存在，因为 Go 中 map 实现中元素的地址是变化的，这意味着寻址的结果是无意义的。

- 为什么 slice 不管是否可寻址，它的元素读是可以寻址的？

>因为 slice 底层实现了一个数组，它是可以寻址的。

- 为什么字符串中的字符/字节又不能寻址呢？

>因为字符串是不可变的。

规范中还有几处提到了 addressable:

- 调用一个接收者为指针类型的方法时，使用一个可寻址的值将自动获取这个值的指针
- ++、-- 语句的操作对象必须可寻址或者是 map 的索引操作
- 赋值语句 = 的左边对象必须可寻址，或者是 map 的索引操作，或者是 _
- 上条同样使用 for ... range 语句
	
参考资料

- [Spec: Address operators](https://golang.org/ref/spec#Address_operators)
- [“cannot take the address of” and “cannot call pointer method on” - stackoverflow](https://stackoverflow.com/questions/44543374/cannot-take-the-address-of-and-cannot-call-pointer-method-on)
- [go addressable 详解](https://colobu.com/2018/02/27/go-addressable/)