title: Golang之内存使用报告
date: 2017-07-04 14:55:54
tags: [Go]
---
Golang 的 runtime 包可以用来检测内存的使用情况，主要内存使用情况，都在 MemStats 结构体里面

<!-- more -->
```go
type MemStats struct {
    // 常用数据   
    Alloc      uint64 // 系统分配了，并且仍在使用的内存
    TotalAlloc uint64 // 分配的内存总量
    Sys        uint64 // 从系统得到的内存总量
    Lookups    uint64 // 回环指针数量
    Mallocs    uint64 // 分配次数
    Frees      uint64 // 内存释放次数

    // 主要的堆数据
    HeapAlloc    uint64 // 分配了之后，并且仍在使用的堆内存
    HeapSys      uint64 // 从系统申请的堆内存大小
    HeapIdle     uint64 // 闲置状态的的span
    HeapInuse    uint64 // 非限制状体的span  HeapIdle + HeapInuse = HeapSys
    HeapReleased uint64 // 从系统中释放的内存大小
    HeapObjects  uint64 // 一共分配的对象数量

    //底层的固定分配数据
    //	按字节计算容量
    //	Sys is bytes obtained from system.
    StackInuse  uint64 // 栈分配使用的内存
    StackSys    uint64  //系统帐使用的内存量
    MSpanInuse  uint64 // mspan结构 使用的量
    MSpanSys    uint64
    MCacheInuse uint64 // mcache 结构使用的量
    MCacheSys   uint64  //系统mcache结构使用的量
    BuckHashSys uint64 // profiling bucket hash table,系统hash表使用情况
    GCSys       uint64 // GC 的元数据
    OtherSys    uint64 // 其他的系统分配数据

    // Garbage collector statistics.
    NextGC       uint64 // 当 HeapAlloc 大于该值的时候，会进行垃圾回收
    LastGC       uint64 // 上一次垃圾回收的时间
    PauseTotalNs uint64
    PauseNs      [256]uint64 // circular buffer of recent GC pause durations, most recent at [(NumGC+255)%256]
    PauseEnd     [256]uint64 // circular buffer of recent GC pause end times
    NumGC        uint32
    EnableGC     bool
    DebugGC      bool

    // Per-size allocation statistics.
    // 61 is NumSizeClasses in the C code.
    BySize [61]struct {
        Size    uint32
        Mallocs uint64
        Frees   uint64
    }
}
```
