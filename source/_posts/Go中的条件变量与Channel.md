title: Go中的条件变量与Channel
date: 2019-07-03 00:04:51
tags: [Go]
---

在Go语言中 `sync.Cond` 代表条件变量，初始化的时候需要传入一个互斥体，它可以是普通锁（Mutex），也可以是读写锁（RWMutex）。如下：

```go
var mutex sync.Mutex  // 也可以是 sync.RWMutex
var cond = sync.NewCond(&mutex)
...
```

<!-- more -->
为什么创建条件变量需要传入锁？因为 `cond.Wait()` 的需要。 Wait 内部实现逻辑是：

```go
把自己加入到挂起队列
mutex.Unlock()
等待被唤醒  // 挂起的执行体会被后续的 cond.Broadcast 或 cond.Signal() 唤醒
mutex.Lock()
```

使用方式：

```go
mutex.Lock()
defer mutex.Unlock()
for conditionNotMetToDo {
    cond.Wait()
}
doSomething
if conditionNeedNotify {
    cond.Broadcast()
    // 有时可以优化为 cond.Signal()
}
```
加锁后，先用一个 for 循环判断当前是否能够做我们想做的事情，如果做不了就调用 cond.Wait() 进行等待。这里很重要的一个细节是注意用的是 for 循环，而不是 if 语句。这是因为 cond.Wait() 得到了执行权后不代表我们想做的事情就一定能够干了，所以要再重新判断一次条件是否满足。

确定能够做事情了，于是 doSomething。在做的过程中，如果我们判断可能挂起队列中的部分执行体满足了重新执行的条件，就用 cond.Broadcast 或 cond.Signal 唤醒它们。

cond.Broadcast 唤醒所有在这个条件变量挂起的执行体，而 cond.Signal 则只唤醒其中一个。

cond.Signal 的适用范围：

- 挂起在这个条件变量上的执行体，它们等待的条件是一致的
- 本次 doSomething 操作完成后，所释放的资源只够一个执行体来做事情

eg：

```go

package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {

	runtime.GOMAXPROCS(4)

	test333()
}

func testCond() {

	c := sync.NewCond(&sync.Mutex{})
	condition := false

	go func() {
		time.Sleep(time.Second * 1)
		c.L.Lock()
		fmt.Println("[1] 变更condition状态,并发出变更通知.")
		condition = true
		c.Signal() //c.Broadcast()
		fmt.Println("[1] 继续后续处理.")
		c.L.Unlock()

	}()

	c.L.Lock()
	fmt.Println("[2] condition..........1")
	for !condition {
		fmt.Println("[2] condition..........2")
		//等待Cond消息通知
		c.Wait()
		fmt.Println("[2] condition..........3")
	}
	fmt.Println("[2] condition..........4")
	c.L.Unlock()

	fmt.Println("main end...")
}

/*

testCond()运行结果:

[2] condition..........1
[2] condition..........2
[1] 变更condition状态,并发出变更通知.
[1] 继续后续处理.
[2] condition..........3
[2] condition..........4
main end...


*/
```
实现一个 channel：

```go
type Channel struct {
    mutex sync.Mutex
    cond *sync.Cond
    queue *Queue
    n int
}

func NewChannel(n int) *Channel {
    if n < 1 {
        panic("todo: support unbuffered channel")
    }
    c := new(Channel)
    c.cond = sync.NewCond(&c.mutex)
    c.queue = NewQueue()
    // 这里 NewQueue 得到一个普通的队列
    // 代码从略
    c.n = n
    return c
}

func (c *Channel) Push(v interface{}) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    for c.queue.Len() == c.n { // 等待队列不满
        c.cond.Wait()
    }
    if c.queue.Len() == 0 { // 原来队列是空的，可能有人等待数据，通知它们
        c.cond.Broadcast()
    }
    c.queue.Push(v)
}

func (c *Channel) Pop() (v interface{}) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    for c.queue.Len() == 0 { // 等待队列不空
        c.cond.Wait()
    }
    if c.queue.Len() == c.n { // 原来队列是满的，可能有人等着写数据，通知它们
        c.cond.Broadcast()
    }
    return c.queue.Pop()
}

func (c *Channel) TryPop() (v interface{}, ok bool) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    if c.queue.Len() == 0 { // 如果队列为空，直接返回
        return
    }
    if c.queue.Len() == c.n { // 原来队列是满的，可能有人等着写数据，通知它们
        c.cond.Broadcast()
    }
    return c.queue.Pop(), true
}

func (c *Channel) TryPush(v interface{}) (ok bool) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    if c.queue.Len() == c.n { // 如果队列满，直接返回
        return
    }
    if c.queue.Len() == 0 { // 原来队列是空的，可能有人等待数据，通知它们
        c.cond.Broadcast()
    }
    c.queue.Push(v)
    return true
}

```

**A线程notify或者signal，被唤醒的线程并不会马上执行，而是需要等待A线程退出同步块或者unlock才会执行**