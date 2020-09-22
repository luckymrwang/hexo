title: Go标准库flag包
date: 2020-09-16 23:14:51
tags: [Go]
---

#### flag包概述

flag包实现了命令行参数的解析

#### flag包的工作流程

基本分为三步：

- 注册flag,主要用flag.String(), Bool(), Int()等函数注册flag，或者用flag.BoolVar()，StringVar(),IntVar()等函数把flag绑定到变量
- 解析flag,使用flag.Parse()函数解析在命令行使用了的flag
- 最后根据命令行输入的flag处理逻辑

<!-- more -->
```go
package main

import (
    "flag"
    "fmt"
    "os"
)

var (

    // 1. 使用flag.String(), Bool(), Int()等函数注册flag,解析后保存到bool,int,string类型的指针
    n = flag.Int("n", 1, "print times .")
    s = flag.String("s", "hello world", "print strings.")

    // 2. 也可以先声明变量，再用flag.BoolVar把flag绑定到变量
    v bool
)

func init() {
    // 3. 绑定flag到变量
    flag.BoolVar(&v, "v", false, "print version and exit.")

    // 4. flag.usage是一个函数变量，在解析flag发生错误时被调用，输出一个说明文本到命令行，可以重写usage的内容
    flag.Usage = usage
}

func main() {

    // 5. 解析命令行flag，这是使用命令行参数之前首先要做的事
    flag.Parse()

    // 6. bool类型flag可以接收的值：1, 0, t, f, T, F, true, false, TRUE, FALSE, True, False
    //    bool类型也可以只写flag名，省略赋值部分，等价与赋值为true."例如：cmd -h "
    if v {
        fmt.Fprintln(os.Stderr, "flag demo version: 1.0.0")
        os.Exit(0)
    }

    // 7. flag.Lookup从命令行flag中返回指定名称的flag，如果没有则返回nil
    if flag.Lookup("s")!=nil {
        if *s != "" {
            for i := 0; i < *n; i++ {

                // 8. 命令行flag语法，可以使用1个或2个'-'号，效果是一样的。
                //    -flag
                //    -flag=x
                //    -flag x  // 只有非bool类型的flag可以
                fmt.Fprintln(os.Stdout, *s)
            }
        }
    }

    // 9. flag.NFlag返回在命令行使用的flag数
    if flag.NFlag() == 0 {
        flag.Usage()
    }

    // 10. visit,遍历所有在命令行中使用的flag，执行自定义函数func
    flag.Visit(func(f *flag.Flag) {
        fmt.Fprintln(os.Stdout,f.Name,f.Value)
    })

    // 11. visitAll,遍历所有注册过的flag，执行自定义函数func
    flag.VisitAll(func(f *flag.Flag) {
        fmt.Fprintln(os.Stdout,f.Name,f.Value)
    })

    // 12. 当flag -help或 -h 被调用但没有注册这个flag时，就会返回flag.usage。
    //    ErrHelp is the error returned if the -help or -h flag is invoked

}

func usage() {
    fmt.Fprint(os.Stderr, `
Usage:flagDemo [options]

Options:

`)
    flag.PrintDefaults()
}
```

####Flag

注册flag就是初始化这个Flag

```go
// A Flag represents the state of a flag.
type Flag struct {
    Name     string // name as it appears on command line
    Usage    string // help message
    Value    Value  // value as set
    DefValue string // default value (as text); for usage message
}
```

#### FlagSet

```go
// A FlagSet represents a set of defined flags. The zero value of a FlagSet
// has no name and has ContinueOnError error handling.
//
// Flag names must be unique within a FlagSet. An attempt to define a flag whose
// name is already in use will cause a panic.
type FlagSet struct {
    // Usage is the function called when an error occurs while parsing flags.
    // The field is a function (not a method) that may be changed to point to
    // a custom error handler. What happens after Usage is called depends
    // on the ErrorHandling setting; for the command line, this defaults
    // to ExitOnError, which exits the program after calling Usage.
    Usage func()

    name          string
    parsed        bool
    actual        map[string]*Flag
    formal        map[string]*Flag
    args          []string // arguments after flags
    errorHandling ErrorHandling
    output        io.Writer // nil means stderr; use out() accessor
}
```

在命令行中使用的所有flag会被解析成一个FlagSet,
我们可以直接使用flag包的Bool(),BoolVar,Parse(),Nflag(),Usage,Visit()等函数去处理flag，这是因为flag包中已经定义了一个FlagSet，名为CommandLine。
我们也可以用NewFlagSet()自己定义一个新的FlagSet去处理命令行flag

区别在于，CommandLine会在Parse()发生错误时退出，而NewFlagSet()可以自定义ErrorHandling

```go
// CommandLine is the default set of command-line flags, parsed from os.Args.
// The top-level functions such as BoolVar, Arg, and so on are wrappers for the
// methods of CommandLine.
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
```

```go
// NewFlagSet returns a new, empty flag set with the specified name and
// error handling property. If the name is not empty, it will be printed
// in the default usage message and in error messages.
func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
    f := &FlagSet{
        name:          name,
        errorHandling: errorHandling,
    }
    f.Usage = f.defaultUsage
    return f
}
```

```go
func (f *FlagSet) Parse(arguments []string) error {
    f.parsed = true
    f.args = arguments
    for {
        seen, err := f.parseOne()
        if seen {
            continue
        }
        if err == nil {
            break
        }
        switch f.errorHandling {
        case ContinueOnError:
            return err
        case ExitOnError:
            os.Exit(2)
        case PanicOnError:
            panic(err)
        }
    }
    return nil
}
```

#### 方法介绍

Bool(),String(),Int(),BoolVar,StringVar(),IntVar(),Var()都是注册flag的方法
Parse()是从命令行解析flag的方法
Lookup(name string),判断是否使用了这个flag
NFlag(),返回命令行使用的flag数
Visit(),按照字母顺序遍历所有在命令行使用的flag
VisitAll(),按照字母顺序遍历所有注册过的flag
PrintDefaults(),打印所有已定义参数的默认值，默认输出到标准错误。（用 VisitAll 实现）

#### 自定义 Value

只要实现 flag.Value 接口即可（要求 receiver 是指针），这时候可以这样注册flag：

```go
flag.Var(&flagVal, "name", "help message for flagname")

// 1. 定义一个类型，用于增加该类型方法
type sliceValue []string

// 2. 实现flag包中的Value接口，将命令行接收到的值用,分隔存到slice里
func (s *sliceValue) Set(val string) error {
    *s = strings.Split(val, ",")
    return nil
}

func (s *sliceValue) String() string {
    return strings.Join(*s, ",")
}

var sv *sliceValue

func init() {
    // 3. 分配内存
    sv = new(sliceValue)
    // 4. 自定义类型绑定到flag
    flag.Var(sv, "sv", "custom sliceValue")
}
```

扩展：

- [github.com/urfave/cli](https://github.com/urfave/cli)
- [github.com/spf13/cobra](https://github.com/spf13/cobra)