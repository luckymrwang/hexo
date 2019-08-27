title: 'Golang 跳转语句goto,break,continue的使用及区别'
date: 2019-08-27 09:21:09
tags: [Golang]
---

### goto

goto 语句可以无条件地转移到过程中指定的行。

通常与条件语句配合使用。可用来实现条件转移，构成循环，跳出循环体等功能。

在结构化程序设计中一般不主张使用goto语句，以免造成程序流程的混乱。

goto对应(标签)既可以定义在for循环前面,也可以定义在for循环后面，当跳转到标签地方时，继续执行标签下面的代码。

```go
func main() {
    //  放在for前面，此例会一直循环下去
    Loop:
    fmt.Println("test")
    for a:=0;a<5;a++{
        fmt.Println(a)
        if a>3{
            goto Loop
        }
    }
}

func main() {
    for a:=0;a<5;a++{
        fmt.Println(a)
        if a>3{
            goto Loop
        }
    }
    Loop:           //放在for后边
    fmt.Println("test")
}
```
<!-- more -->

### break

```go
func main() {
    Loop:
    for j:=0;j<3;j++{
        fmt.Println(j)
        for a:=0;a<5;a++{
            fmt.Println(a)
            if a>3{
                break Loop
            }
        }
    }
}
```

### continue

```go
continue和标签的使用类似于break，这里不再详述
```

### 总结

```goto语句本身就是做跳转用的，而break和continue是配合for使用的。所以goto的使用不限于for，通常与条件语句配合使用
在for循环中break和continue可以配合标签使用。
```
