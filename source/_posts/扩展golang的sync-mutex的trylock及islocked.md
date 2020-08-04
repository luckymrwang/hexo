title: 扩展golang的sync mutex的trylock及islocked
date: 2020-02-11 16:24:31
tags: [Go]
---

  Golang的 sync mutex是阻塞互斥锁，在一个 goroutine 获得 Mutex 后，其他 goroutine 只能等到这个 goroutine 释放该 Mutex，使用 Lock() 加锁后，不能再继续对其加锁，直到利用 Unlock() 解锁后才能再加锁。但有时逻辑里面需要 非阻塞模式的拿锁 及 非阻塞得知锁的状态。trylock, 可以用非阻塞的模型进行拿锁，要么拿到锁，要么锁被别人拿到。 islocked, 就是判断锁的状态。
  
<!-- more -->

  实现的逻辑相对简单，就是使用golang atomic标准库做compareAndSet原子更新,  如果更新成功为拿到锁，否则，反之。  cas 底层是依赖cpu的指令集的，cas的操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。 如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值 。否则，处理器不做任何操作。
  
```go
package trylock

import (
	"sync"
	"sync/atomic"
	"unsafe"
)

const (
	LockedFlag   int32 = 1
	UnlockedFlag int32 = 0
)

type Mutex struct {
	in     sync.Mutex
	status *int32
}

func NewMutex() *Mutex {
	status := UnlockedFlag
	return &Mutex{
		status: &status,
	}
}

func (m *Mutex) Lock() {
	m.in.Lock()
}

func (m *Mutex) Unlock() {
	m.in.Unlock()
	atomic.AddInt32(m.status, UnlockedFlag)
}

func (m *Mutex) TryLock() bool {
	if atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.in)), UnlockedFlag, LockedFlag) {
		atomic.AddInt32(m.status, LockedFlag)
		return true
	}
	return false
}

func (m *Mutex) IsLocked() bool {
	if atomic.LoadInt32(m.status) == LockedFlag {
		return true
	}
	return false
}
```

golang sync mutex性能很高，因为他底层也是cas + go runtime 调度实现的。

本文引自[这里](http://xiaorui.cc/2018/02/27/%E6%89%A9%E5%B1%95golang%E7%9A%84sync-mutex%E7%9A%84trylock%E5%8F%8Aislocked/)