title: Golang内存分配源码解析
date: 2022-05-08 10:09:01
tags: [Golang]
---

### Golang内存分配整体架构

![image](/images/golang-memory1.jpg)

<!-- more -->
简化后：

![image](/images/golang-memory2.jpg)

### makeslice

以创建切片为例，在64位操作系统上，会调用`makeslice64()`，最终都会去调用`makeslice()`去进行最终的创建动作，先计算是否内存溢出或越界，再通过调用`mallocgc`分配内存给`slice`，源码在 runtime/slice.go

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))	
	if overflow || mem > maxAlloc || len < 0 || len > cap {	//判断内存是否溢出或者需要的内存大于系统可以给的最大内存
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()	//len越界
		}
		panicmakeslicecap()//cap越界
	}

	return mallocgc(mem, et, true)//申请内存
}

func makeslice64(et *_type, len64, cap64 int64) unsafe.Pointer {//64位添加保护机制，判断申请的大小是否超出或小于阈值
	len := int(len64)
	if int64(len) != len64 {
		panicmakeslicelen()
	}

	cap := int(cap64)
	if int64(cap) != cap64 {
		panicmakeslicecap()
	}

	return makeslice(et, len, cap)
}
```
### 内存初始化-mallocinit

- 检查系统/硬件信息
- 计算预留空间大小
- 尝试预留地址
- 初始化mheap中的一部分变量
- 其他部分初始化，67个mcentral在这里初始化

#### mcentral初始化：

- 设置自己的级别
- 将两个mspanList初始化

```go
func mallocinit() {
	if class_to_size[_TinySizeClass] != _TinySize { // 校tiny分配大小
		throw("bad TinySizeClass")
	}

	testdefersizes()

	if heapArenaBitmapBytes&(heapArenaBitmapBytes-1) != 0 {
		// heapBits expects modular arithmetic on bitmap
		// addresses to work.
		throw("heapArenaBitmapBytes not a power of 2")
	}

	// Copy class sizes out for statistics table.
	for i := range class_to_size {
		memstats.by_size[i].size = uint32(class_to_size[i])
	}

	// Check physPageSize.
	if physPageSize == 0 { // 检查物理页大小
		// The OS init code failed to fetch the physical page size.
		throw("failed to get system page size")
	}
	if physPageSize > maxPhysPageSize {
		print("system page size (", physPageSize, ") is larger than maximum page size (", maxPhysPageSize, ")\n")
		throw("bad system page size")
	}
	if physPageSize < minPhysPageSize {
		print("system page size (", physPageSize, ") is smaller than minimum page size (", minPhysPageSize, ")\n")
		throw("bad system page size")
	}
	if physPageSize&(physPageSize-1) != 0 {
		print("system page size (", physPageSize, ") must be a power of 2\n")
		throw("bad system page size")
	}
	if physHugePageSize&(physHugePageSize-1) != 0 {
		print("system huge page size (", physHugePageSize, ") must be a power of 2\n")
		throw("bad system huge page size")
	}
	if physHugePageSize > maxPhysHugePageSize {
		// physHugePageSize is greater than the maximum supported huge page size.
		// Don't throw here, like in the other cases, since a system configured
		// in this way isn't wrong, we just don't have the code to support them.
		// Instead, silently set the huge page size to zero.
		physHugePageSize = 0
	}
	if physHugePageSize != 0 {
		// Since physHugePageSize is a power of 2, it suffices to increase
		// physHugePageShift until 1<<physHugePageShift == physHugePageSize.
		for 1<<physHugePageShift != physHugePageSize {
			physHugePageShift++
		}
	}
	if pagesPerArena%pagesPerSpanRoot != 0 {
		print("pagesPerArena (", pagesPerArena, ") is not divisible by pagesPerSpanRoot (", pagesPerSpanRoot, ")\n")
		throw("bad pagesPerSpanRoot")
	}
	if pagesPerArena%pagesPerReclaimerChunk != 0 {
		print("pagesPerArena (", pagesPerArena, ") is not divisible by pagesPerReclaimerChunk (", pagesPerReclaimerChunk, ")\n")
		throw("bad pagesPerReclaimerChunk")
	}

	// Initialize the heap.
	mheap_.init() // 初始化堆内存
	mcache0 = allocmcache()
	lockInit(&gcBitsArenas.lock, lockRankGcBitsArenas)
	lockInit(&proflock, lockRankProf)
	lockInit(&globalAlloc.mutex, lockRankGlobalAlloc)

	// Create initial arena growth hints.
  // 初始化内存分配 arena，arena 是一段连续的内存，负责数据的内存分配。
	if sys.PtrSize == 8 { // 64位不同系统上进行地址的划分
		// On a 64-bit machine, we pick the following hints
		// because:
		//
		// 1. Starting from the middle of the address space
		// makes it easier to grow out a contiguous range
		// without running in to some other mapping.
		//
		// 2. This makes Go heap addresses more easily
		// recognizable when debugging.
		//
		// 3. Stack scanning in gccgo is still conservative,
		// so it's important that addresses be distinguishable
		// from other data.
		//
		// Starting at 0x00c0 means that the valid memory addresses
		// will begin 0x00c0, 0x00c1, ...
		// In little-endian, that's c0 00, c1 00, ... None of those are valid
		// UTF-8 sequences, and they are otherwise as far away from
		// ff (likely a common byte) as possible. If that fails, we try other 0xXXc0
		// addresses. An earlier attempt to use 0x11f8 caused out of memory errors
		// on OS X during thread allocations.  0x00c0 causes conflicts with
		// AddressSanitizer which reserves all memory up to 0x0100.
		// These choices reduce the odds of a conservative garbage collector
		// not collecting memory because some non-pointer block of memory
		// had a bit pattern that matched a memory address.
		//
		// However, on arm64, we ignore all this advice above and slam the
		// allocation at 0x40 << 32 because when using 4k pages with 3-level
		// translation buffers, the user address space is limited to 39 bits
		// On ios/arm64, the address space is even smaller.
		//
		// On AIX, mmaps starts at 0x0A00000000000000 for 64-bit.
		// processes.
		for i := 0x7f; i >= 0; i-- {
			var p uintptr
			switch {
			case raceenabled:
				// The TSAN runtime requires the heap
				// to be in the range [0x00c000000000,
				// 0x00e000000000).
				p = uintptr(i)<<32 | uintptrMask&(0x00c0<<32)
				if p >= uintptrMask&0x00e000000000 {
					continue
				}
			case GOARCH == "arm64" && GOOS == "ios":
				p = uintptr(i)<<40 | uintptrMask&(0x0013<<28)
			case GOARCH == "arm64":
				p = uintptr(i)<<40 | uintptrMask&(0x0040<<32)
			case GOOS == "aix":
				if i == 0 {
					// We don't use addresses directly after 0x0A00000000000000
					// to avoid collisions with others mmaps done by non-go programs.
					continue
				}
				p = uintptr(i)<<40 | uintptrMask&(0xa0<<52)
			default:
				p = uintptr(i)<<40 | uintptrMask&(0x00c0<<32)
			}
			hint := (*arenaHint)(mheap_.arenaHintAlloc.alloc())
			hint.addr = p
			hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
		}
	} else { //32 位
		// On a 32-bit machine, we're much more concerned
		// about keeping the usable heap contiguous.
		// Hence:
		//
		// 1. We reserve space for all heapArenas up front so
		// they don't get interleaved with the heap. They're
		// ~258MB, so this isn't too bad. (We could reserve a
		// smaller amount of space up front if this is a
		// problem.)
		//
		// 2. We hint the heap to start right above the end of
		// the binary so we have the best chance of keeping it
		// contiguous.
		//
		// 3. We try to stake out a reasonably large initial // 栈预留
		// heap reservation.

		const arenaMetaSize = (1 << arenaBits) * unsafe.Sizeof(heapArena{})
		meta := uintptr(sysReserve(nil, arenaMetaSize))
		if meta != 0 {
			mheap_.heapArenaAlloc.init(meta, arenaMetaSize)
		}

		// 我们想要让 arena 区域从低地址开始，但是我们的代码可能和 C 代码进行链接，
        // 全局的构造器可能已经调用过 malloc，并且调整过进程的 brk 位置。
        // 所以需要查询一次 brk，以避免将我们的 arena 区域覆盖掉 brk 位置，
        // 这会导致 kernel 把 arena 放在其它地方，比如放在高地址。
		procBrk := sbrk0() 
    
    //Linux系统分配用户内存的API是brk()。它的作用就是改变数据段的末尾位置，也就是改变进程的brk的位置。
    //进程的brk之前的内存，就是可以使用的堆内存。移动brk的位置，就可以分配内存。操作系统的API就是这么简单直接。
    //brk()函数，是Linux的一个系统调用。在x86_64平台上，它的调用号是12。

		// If we ask for the end of the data segment but the
		// operating system requires a little more space
		// before we can start allocating, it will give out a
		// slightly higher pointer. Except QEMU, which is
		// buggy, as usual: it won't adjust the pointer
		// upward. So adjust it upward a little bit ourselves:
		// 1/4 MB to get away from the running binary image.
		p := firstmoduledata.end
		if p < procBrk {
			p = procBrk
		}
		if mheap_.heapArenaAlloc.next <= p && p < mheap_.heapArenaAlloc.end {
			p = mheap_.heapArenaAlloc.end
		}
		p = alignUp(p+(256<<10), heapArenaBytes)
		// Because we're worried about fragmentation on
		// 32-bit, we try to make a large initial reservation.
    //// 如果分配失败，那么尝试用更小一些的 arena 区域。
        // 对于像 Android L 这样的系统是需要的，因为我们和 ART 更新同一个进程，
        // 其会更激进地保留内存。
        // 最差的情况下，会退化为 0 大小的初始 arena
        // 这种情况下希望之后紧跟着的内存保留操作能够成功。
    
		arenaSizes := []uintptr{
			512 << 20,
			256 << 20,
			128 << 20,
		}
		for _, arenaSize := range arenaSizes {
      // 申请初始化对齐内存
			a, size := sysReserveAligned(unsafe.Pointer(p), arenaSize, heapArenaBytes)
			if a != nil {
				mheap_.arena.init(uintptr(a), size) // 初始化arena
				p = mheap_.arena.end // For hint below
				break
			}
		}
		hint := (*arenaHint)(mheap_.arenaHintAlloc.alloc())
		hint.addr = p
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
	}
}
```

`mcache`的初始化在`func procresize(nprocs int32)`中，`procresize`也在`schedinit()`中调用，顺序在`mallocinit()`之后，所以说也就是说`mcentral`先初始化，然后是`mcache`，它在初始化P的时候初始化具体代码如下：

```go
func allocmcache() *mcache {
	var c *mcache
	systemstack(func() {
		lock(&mheap_.lock)
		c = (*mcache)(mheap_.cachealloc.alloc())
		c.flushGen = mheap_.sweepgen
		unlock(&mheap_.lock)
	})
	for i := range c.alloc {
		c.alloc[i] = &emptymspan
	}
	c.nextSample = nextSample()
	return c
}
```

这个初始化后，管理结构、mheap、67个mcentral及每个goroutine的mcache都初始化完毕。接下来就是使用时，如何分配和管理内存。

#### mallocgc-内存申请

/runtime/malloc.go

```go
// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if gcphase == _GCmarktermination { //判断是否正在GC，如果在GC，等待GC完后再拉起用户协程继续（stop the world）
		throw("mallocgc called with gcphase == _GCmarktermination")
	}

  if size == 0 { //zerobase:所有0字节的基地址（空指针）
		return unsafe.Pointer(&zerobase)
	}

	if debug.malloc {
		if debug.sbrk != 0 { //sbrk=1时会使用一个碎片回收器代替内存分配器和垃圾回收器。它从操作系统获取内存，并且永远也不会回收任何内存。用于调试，正常流程不会走这段
			align := uintptr(16)
			if typ != nil {//内存对齐
				// TODO(austin): This should be just
				//   align = uintptr(typ.align)
				// but that's only 4 on 32-bit platforms,
				// even if there's a uint64 field in typ (see #599).
				// This causes 64-bit atomic accesses to panic.
				// Hence, we use stricter alignment that matches
				// the normal allocator better.
				if size&7 == 0 {
					align = 8
				} else if size&3 == 0 {
					align = 4
				} else if size&1 == 0 {
					align = 2
				} else {
					align = 1
				}
			}
			return persistentalloc(size, align, &memstats.other_sys)
		}

		if inittrace.active && inittrace.id == getg().goid { //初始化trace的统计信息，trace信息就是在这加载的
			// Init functions are executed sequentially in a single Go routine.
			inittrace.allocs += 1
		}
	}

	// assistG is the G to charge for this allocation, or nil if
	// GC is not currently active.
  //为了保证用户程序分配内存的速度不会超出后台任务的标记速度，runtime引入了标记辅助技术，它遵循一条非常简单并且朴实的原则，分配多少内存就需要完成多少标记任务。每一个 Goroutine 都持有 gcAssistBytes 字段，这个字段存储了当前 Goroutine 辅助标记的对象字节数。在并发标记阶段期间，当 Goroutine 调用 runtime.mallocgc 分配新对象时，该函数会检查申请内存的 Goroutine 是否处于入不敷出的状态
	var assistG *g
	if gcBlackenEnabled != 0 {
		// Charge the current user G for this allocation.
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		// Charge the allocation against the G. We'll account
		// for internal fragmentation at the end of mallocgc.
		assistG.gcAssistBytes -= int64(size)

		if assistG.gcAssistBytes < 0 { //会按分配的大小判断需要协助GC完成多少工作
			// This G is in debt. Assist the GC to correct
			// this before allocating. This must happen
			// before disabling preemption.
      //申请内存时调用的 runtime.gcAssistAlloc 和扫描内存时调用的 runtime.gcFlushBgCredit 分别负责借债和还债，通过这套债务管理系统，保证 Goroutine 在正常运行的同时不会为垃圾收集造成太多的压力，保证在达到堆大小目标时完成标记阶段。
			gcAssistAlloc(assistG)
		}
	}

	// Set mp.mallocing to keep from being preempted by GC.
	mp := acquirem() //设置 mp.mallocing 以防止被 GC 抢占。（独占m）
	if mp.mallocing != 0 { //如果m正在分配内存，则造成死锁
		throw("malloc deadlock")
	}
	if mp.gsignal == getg() {
		throw("malloc during signal")
	}
	mp.mallocing = 1 //标记为正在分配内存

	shouldhelpgc := false
	dataSize := size
	c := getMCache() //获取当前m的m mcache
	if c == nil {
		throw("mallocgc called without a P or outside bootstrapping")
	}
	var span *mspan
	var x unsafe.Pointer
	noscan := typ == nil || typ.ptrdata == 0 //判断是否是指针类型
	if size <= maxSmallSize { //小于等于32KB的对象申请内存
		if noscan && size < maxTinySize { //非指针（值类型）&&小于16b的内存，使用tiny分配
			// Tiny allocator.
			//
			// Tiny allocator combines several tiny allocation requests
			// into a single memory block. The resulting memory block
			// is freed when all subobjects are unreachable. The subobjects
			// must be noscan (don't have pointers), this ensures that
			// the amount of potentially wasted memory is bounded.
			//
			// Size of the memory block used for combining (maxTinySize) is tunable.
			// Current setting is 16 bytes, which relates to 2x worst case memory
			// wastage (when all but one subobjects are unreachable).
			// 8 bytes would result in no wastage at all, but provides less
			// opportunities for combining.
			// 32 bytes provides more opportunities for combining,
			// but can lead to 4x worst case wastage.
			// The best case winning is 8x regardless of block size.
			//
			// Objects obtained from tiny allocator must not be freed explicitly.
			// So when an object will be freed explicitly, we ensure that
			// its size >= maxTinySize.
			//
			// SetFinalizer has a special case for objects potentially coming
			// from tiny allocator, it such case it allows to set finalizers
			// for an inner byte of a memory block.
			//
			// The main targets of tiny allocator are small strings and
			// standalone escaping variables. On a json benchmark
			// the allocator reduces number of allocations by ~12% and
			// reduces heap size by ~20%.
      //tiny allocator的主要目标是小字符串和独立的转义变量。在 json 基准测试中分配器将分配次数减少了约 12% 并且将堆大小减少约 20%
			off := c.tinyoffset //取出mcache中tiny区域上次用到的偏移量
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 { //内存对齐
				off = alignUp(off, 8) //往上四舍五入到8的倍数，让每个变量从2的n次方处开始分配
			} else if sys.PtrSize == 4 && size == 12 {
				// Conservatively align 12-byte objects to 8 bytes on 32-bit
				// systems so that objects whose first field is a 64-bit
				// value is aligned to 8 bytes and does not cause a fault on
				// atomic access. See issue 37262.
				// TODO(mknyszek): Remove this workaround if/when issue 36606
				// is resolved.
				off = alignUp(off, 8)
			} else if size&3 == 0 {
				off = alignUp(off, 4)
			} else if size&1 == 0 {
				off = alignUp(off, 2)
			}
			if off+size <= maxTinySize && c.tiny != 0 { //如果此次分配后的偏移<=16，同时tiny还有可用空间，则直接使用mcache的tiny内存块分配
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off) //得到变量开始位置（区域应该是c.tiny + off ~ c.tiny + off + size）
				c.tinyoffset = off + size //更新偏移量
				c.tinyAllocs++ //记录分配的tiny对象的总个数
				mp.mallocing = 0 //标记为分配结束
				releasem(mp) //放弃独占m
				return x //返回变量的开始位置
			}
			// Allocate a new maxTinySize block.
      //没有可用的tiny对象给分配时：
			span = c.alloc[tinySpanClass] //向mcache申请
			v := nextFreeFast(span) // 得到一个tinySpanClass规格的span的可用区域的开始位置，先从allocCache中看是否有足够空闲的elem快速分配
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass) //没有足够的缓存空闲对象可用时，从中心缓存或者页堆中获取新的管理单元，在这时就可能触发垃圾收集；调用refill() （后文会展开详解），shouldhelpgc会等于true时会在下面判断是否要触发GC
			}
			x = unsafe.Pointer(v) //x是这个函数的返回值，v设置为返回值，也就是v就是变量开始位置
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {	//看是否要用新的tiny小块替换原来的tiny块
				c.tiny = uintptr(x) //设置新的tiny块
				c.tinyoffset = size //设置新的偏移量
			}
			size = maxTinySize
		} else { //分配的内存>16byte且<= 32k，或者是指针类型变量 : has pointer(scan) || (size >= 16bytes && size <= 32KB)
			var sizeclass uint8
			if size <= smallSizeMax-8 { //如果目标大小<=1016b，就是下面的规格
				sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
			} else {
				sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
			}
			size = uintptr(class_to_size[sizeclass]) //得到规格的大小
			spc := makeSpanClass(sizeclass, noscan) //得到规格的大小
			span = c.alloc[spc] //取出这个规格的span
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc) //与tiny一样，mcache没有的话会向mcentral申请，mcentral没有会向mheap申请，具体详见refill方法
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
        memclrNoHeapPointers(unsafe.Pointer(v), size) //个人理解是清理被分配的内存(清零)，保证是干净的
			}
		}
	} else { //分配目标>32KB
		shouldhelpgc = true
		span = c.allocLarge(size, needzero, noscan) //从mheap中分配内存
		span.freeindex = 1
		span.allocCount = 1
		x = unsafe.Pointer(span.base())
		size = span.elemsize
	}

	var scanSize uintptr
  //设置arena对应的bitmap, 记录哪些位置包含了指针, GC会使用bitmap扫描所有可到达的对象
	if !noscan { //是指针
		// If allocating a defer+arg block, now that we've picked a malloc size
		// large enough to hold everything, cut the "asked for" size down to
		// just the defer header, so that the GC bitmap will record the arg block
		// as containing nothing at all (as if it were unused space at the end of
		// a malloc block caused by size rounding).
		// The defer arg areas are scanned as part of scanstack.
		if typ == deferType {
			dataSize = unsafe.Sizeof(_defer{})
		}
    //根据编译期间对每个struct生成的type结构，用一个bitmap记录下来分配的内存块中哪些位置是指针。
		heapBitsSetType(uintptr(x), size, dataSize, typ)
		if dataSize > typ.size {
			// Array allocation. If there are any
			// pointers, GC has to scan to the last
			// element.
			if typ.ptrdata != 0 { /// 数组分配。 如果有任何指针，GC 必须扫描到最后元素。
				scanSize = dataSize - typ.size + typ.ptrdata
			}
		} else {
			scanSize = typ.ptrdata
		}
		c.scanAlloc += scanSize
	}

	// Ensure that the stores above that initialize x to
	// type-safe memory and set the heap bits occur before
	// the caller can make x observable to the garbage
	// collector. Otherwise, on weakly ordered machines,
	// the garbage collector could follow a pointer to x,
	// but see uninitialized memory or stale heap bits.
  // 内存屏障, 因为x86和x64的store不会乱序所以这里只是个针对编译器的屏障, 汇编中是ret
	publicationBarrier()

	// Allocate black during GC.
	// All slots hold nil so no scanning is needed.
	// This may be racing with GC so do it atomically if there can be
	// a race marking the bit.
	if gcphase != _GCoff { // 如果当前在GC中, 需要立刻标记分配后的对象为"黑色", 防止它被回收
		gcmarknewobject(span, uintptr(x), size, scanSize)
	}

  // Race Detector的处理(用于检测线程冲突问题)
	if raceenabled {
		racemalloc(x, size)
	}

  // Race Detector的处理(用于检测线程冲突问题)
	if msanenabled {
		msanmalloc(x, size)
	}

	mp.mallocing = 0 //标记结束
	releasem(mp) //放弃独占

  // trace记录
	if debug.malloc {
		if debug.allocfreetrace != 0 {
			tracealloc(x, size, typ)
		}

		if inittrace.active && inittrace.id == getg().goid {
			// Init functions are executed sequentially in a single Go routine.
			inittrace.bytes += uint64(size)
		}
	}

  // Profiler记录
	if rate := MemProfileRate; rate > 0 {
		if rate != 1 && size < c.nextSample {
			c.nextSample -= size
		} else {
			mp := acquirem()
			profilealloc(mp, x, size)
			releasem(mp)
		}
	}

  // gcAssistBytes减去"实际分配大小 - 要求分配大小", 调整到准确值
	if assistG != nil {
		// Account for internal fragmentation in the assist
		// debt now that we know it.
		assistG.gcAssistBytes -= int64(size - dataSize)
	}

	if shouldhelpgc { //如果之前获取了新的span, 则判断是否需要后台启动GC
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}

	return x
}
```

#### 什么是内存对齐？

![image](/images/golang-memory3.jpg)

变量 a、b 各占据 3 字节的空间，内存对齐后，a、b 占据 4 字节空间，CPU 读取 b 变量的值只需要进行一次内存访问。如果不进行内存对齐，CPU 读取 b 变量的值需要进行 2 次内存访问。第一次访问得到 b 变量的第 1 个字节，第二次访问得到 b 变量的后两个字节。如果不进行内存对齐，很可能增加 CPU 访问内存的次数。

#### runtime.nextFreeFast-尝试从缓存中快速分配

```go
// nextFreeFast returns the next free object if one is quickly available.
// Otherwise it returns 0.
func nextFreeFast(s *mspan) gclinkptr {
  // 获取第一个非0的bit是第几个, 查找哪个元素是未分配的
	theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
	if theBit < 64 { // 索引小于元素数量，证明找到了
		result := s.freeindex + uintptr(theBit)
		if result < s.nelems {
			freeidx := result + 1
			if freeidx%64 == 0 && freeidx != s.nelems { // 可以被64整除时需要特殊处理
				return 0
			}
			s.allocCache >>= uint(theBit + 1) // 更新freeindex和allocCache(高位都是0, 用尽以后会更新)
			s.freeindex = freeidx 
      
			s.allocCount++ // 添加已分配的elem计数
			return gclinkptr(result*s.elemsize + s.base()) // 返会分配到的元素地址
		}
	}
	return 0
}
```

#### mcache.nextFree-申请新的span

如果在freeindex后无法快速找到未分配的元素, 就需要调用[nextFree](https://github.com/golang/go/blob/go1.9.2/src/runtime/malloc.go#L546) :

```go
// nextFree returns the next free object from the cached span if one is available.
// Otherwise it refills the cache with a span with an available object and
// returns that object along with a flag indicating that this was a heavy
// weight allocation. If it is a heavy weight allocation the caller must
// determine whether a new GC cycle needs to be started or if the GC is active
// whether this goroutine needs to assist the GC.
//
// Must run in a non-preemptible context since otherwise the owner of
// c could change.
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
	s = c.alloc[spc]
	shouldhelpgc = false
	freeIndex := s.nextFreeIndex()
	if freeIndex == s.nelems { // 如果span里面所有元素都已分配, 则需要获取新的span
		// The span is full.
		if uintptr(s.allocCount) != s.nelems {
			println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
			throw("s.allocCount != s.nelems && freeIndex == s.nelems")
		}
		c.refill(spc) //申请新的span
		shouldhelpgc = true
		s = c.alloc[spc]

		freeIndex = s.nextFreeIndex()
	}

	if freeIndex >= s.nelems {
		throw("freeIndex is not valid")
	}

  // 返会分配到的元素地址
	v = gclinkptr(freeIndex*s.elemsize + s.base())
	s.allocCount++ //更新分配计数
	if uintptr(s.allocCount) > s.nelems {
		println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
		throw("s.allocCount > s.nelems")
	}
	return
}
```

#### mcache.refill-向mcentral申请内存

如果mcache中指定类型的span已满, 就需要调用[refill](https://github.com/golang/go/blob/go1.9.2/src/runtime/mcache.go#L107)函数申请新的span:

```go
// refill acquires a new span of span class spc for c. This span will
// have at least one free object. The current span in c must be full.
//
// Must run in a non-preemptible context since otherwise the owner of
// c could change.
func (c *mcache) refill(spc spanClass) {
	// Return the current cached span to the central lists.
	s := c.alloc[spc]

	if uintptr(s.allocCount) != s.nelems { // 确保所有的span都已经被分配
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// Mark this span as no longer cached.
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		mheap_.central[spc].mcentral.uncacheSpan(s) // 将span归还
	}

	// Get a new cached span from the central lists.
	s = mheap_.central[spc].mcentral.cacheSpan() // 从mcentral中申请新的span
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// Indicate that this span is cached and prevent asynchronous
	// sweeping in the next sweep phase.
	s.sweepgen = mheap_.sweepgen + 3

	// Assume all objects from this span will be allocated in the
	// mcache. If it gets uncached, we'll adjust this.
	stats := memstats.heapStats.acquire()
	atomic.Xadduintptr(&stats.smallAllocCount[spc.sizeclass()], uintptr(s.nelems)-uintptr(s.allocCount))
	memstats.heapStats.release()

	// Update heap_live with the same assumption.
	usedBytes := uintptr(s.allocCount) * s.elemsize
	atomic.Xadd64(&memstats.heap_live, int64(s.npages*pageSize)-int64(usedBytes))

	// Flush tinyAllocs.
	if spc == tinySpanClass {
		atomic.Xadd64(&memstats.tinyallocs, int64(c.tinyAllocs))
		c.tinyAllocs = 0
	}

	// While we're here, flush scanAlloc, since we have to call
	// revise anyway.
	atomic.Xadd64(&memstats.heap_scan, int64(c.scanAlloc))
	c.scanAlloc = 0

	if trace.enabled {
		// heap_live changed.
		traceHeapAlloc()
	}
	if gcBlackenEnabled != 0 {
		// heap_live and heap_scan changed.
		gcController.revise()
	}

	c.alloc[spc] = s // 设置新的span到mcache中
}
```

#### mcentral.cacheSpan-向mcentral申请span

向mcentral申请一个新的span会通过[cacheSpan](https://github.com/golang/go/blob/go1.9.2/src/runtime/mcentral.go#L40)函数:

```go
// Allocate a span to use in an mcache.
func (c *mcentral) cacheSpan() *mspan {
	// Deduct credit for this span allocation and sweep if necessary.
  // 让当前G协助一部分的sweep工作
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	deductSweepCredit(spanBytes, 0)

	sg := mheap_.sweepgen

	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}

	// If we sweep spanBudget spans without finding any free
	// space, just allocate a fresh span. This limits the amount
	// of time we can spend trying to find free space and
	// amortizes the cost of small object sweeping over the
	// benefit of having a full free span to allocate from. By
	// setting this to 100, we limit the space overhead to 1%.
	//
	// TODO(austin,mknyszek): This still has bad worst-case
	// throughput. For example, this could find just one free slot
	// on the 100th swept span. That limits allocation latency, but
	// still has very poor throughput. We could instead keep a
	// running free-to-used budget and switch to fresh span
	// allocation if the budget runs low.
	spanBudget := 100

	var s *mspan
  
  // **新版中的 partial，full对应老版本的nonemptyh和enpty两个链表，partial维护该span最少有一个未分配的元素，full表示不确定该span最少有一个未分配的元素,优先从 partial 中查找

	// Try partial swept spans first.
  // 从清理过的、包含空闲空间的spanSet结构中查找可以使用的内存管理单元
	if s = c.partialSwept(sg).pop(); s != nil {
		goto havespan
	}

	// Now try partial unswept spans.
	for ; spanBudget >= 0; spanBudget-- {
		s = c.partialUnswept(sg).pop() // 从未被清理过的、有空闲对象的spanSet查找可用的span
		if s == nil {
			break
		}
		if atomic.Load(&s.sweepgen) == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// We got ownership of the span, so let's sweep it and use it.
      // 找到要回收的span，触发sweep进行清理
			s.sweep(true)
			goto havespan
		}
		// We failed to get ownership of the span, which means it's being or
		// has been swept by an asynchronous sweeper that just couldn't remove it
		// from the unswept list. That sweeper took ownership of the span and
		// responsibility for either freeing it to the heap or putting it on the
		// right swept list. Either way, we should just ignore it (and it's unsafe
		// for us to do anything else).
	}
	// Now try full unswept spans, sweeping them and putting them into the
	// right list if we fail to get a span.
	for ; spanBudget >= 0; spanBudget-- {
		s = c.fullUnswept(sg).pop() // 获取未被清理的、不包含空闲空间的spanSet查找可用的span
		if s == nil {
			break
		}
		if atomic.Load(&s.sweepgen) == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// We got ownership of the span, so let's sweep it.
			s.sweep(true)
			// Check if there's any free space.
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			// Add it to the swept list, because sweeping didn't give us any free space.
			c.fullSwept(sg).push(s)
		}
		// See comment for partial unswept spans.
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}

	// We failed to get a span from the mcentral so get one from mheap.
  // 从堆中申请新的内存管理单元
	s = c.grow()
	if s == nil {
		return nil
	}

	// At this point s is a span that should have free slots.
havespan:
	if trace.enabled && !traceDone {
		traceGCSweepDone()
	}
	n := int(s.nelems) - int(s.allocCount)
	if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
		throw("span has no free objects")
	}
	freeByteBase := s.freeindex &^ (64 - 1)
	whichByte := freeByteBase / 8
	// Init alloc bits cache.
	s.refillAllocCache(whichByte)  // 更新AllocCache

	// Adjust the allocCache so that s.freeindex corresponds to the low bit in
	// s.allocCache.
	s.allocCache >>= s.freeindex % 64

	return s
}
```

#### mcentral.grow-向heap申请内存

mcentral向mheap申请一个新的span会使用[grow](https://github.com/golang/go/blob/go1.9.2/src/runtime/mcentral.go#L227)函数:

```go
// grow allocates a new empty span from the heap and initializes it for c's size class.func (c *mcentral) grow() *mspan {  // 根据mcentral的类型计算需要申请的span的大小(除以8K = 有多少页)和可以保存多少个元素	npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])	size := uintptr(class_to_size[c.spanclass.sizeclass()])  // 向mheap申请一个新的span, 以页(8K)为单位	s := mheap_.alloc(npages, c.spanclass, true)	if s == nil {		return nil	}	// Use division by multiplication and shifts to quickly compute:	// n := (npages << _PageShift) / size	n := (npages << _PageShift) >> s.divShift * uintptr(s.divMul) >> s.divShift2	s.limit = s.base() + size*n // 更新limit	heapBitsForAddr(s.base()).initSpan(s) // 分配并初始化span的allocBits和gcmarkBits	return s}
```

#### mcache.allocLarge-大对象分配

对于大对象的分配allocLarge，直接回向mheap申请，同mheap_.alloc

```go
// allocLarge allocates a span for a large object.func (c *mcache) allocLarge(size uintptr, needzero bool, noscan bool) *mspan {	if size+_PageSize < size {		throw("out of memory")	}	npages := size >> _PageShift	if size&_PageMask != 0 {		npages++	}	// Deduct credit for this span allocation and sweep if	// necessary. mHeap_Alloc will also sweep npages, so this only	// pays the debt down to npage pages.	deductSweepCredit(npages*_PageSize, npages)	spc := makeSpanClass(0, noscan)	s := mheap_.alloc(npages, spc, needzero)	if s == nil {		throw("out of memory")	}	stats := memstats.heapStats.acquire()	atomic.Xadduintptr(&stats.largeAlloc, npages*pageSize)	atomic.Xadduintptr(&stats.largeAllocCount, 1)	memstats.heapStats.release()	// Update heap_live and revise pacing if needed.	atomic.Xadd64(&memstats.heap_live, int64(npages*pageSize))	if trace.enabled {		// Trace that a heap alloc occurred because heap_live changed.		traceHeapAlloc()	}	if gcBlackenEnabled != 0 {		gcController.revise()	}	// Put the large span in the mcentral swept list so that it's	// visible to the background sweeper.	mheap_.central[spc].mcentral.fullSwept(mheap_.sweepgen).push(s)	s.limit = s.base() + size	heapBitsForAddr(s.base()).initSpan(s)	return s}
```

#### mheap分配流程

##### alloc

```go
// alloc allocates a new span of npage pages from the GC'd heap.//// spanclass indicates the span's size class and scannability.//// If needzero is true, the memory for the returned span will be zeroed.func (h *mheap) alloc(npages uintptr, spanclass spanClass, needzero bool) *mspan {	// Don't do any operations that lock the heap on the G stack.	// It might trigger stack growth, and the stack growth code needs	// to be able to allocate heap.	var s *mspan	systemstack(func() {		// To prevent excessive heap growth, before allocating n pages		// we need to sweep and reclaim at least n pages.		if h.sweepdone == 0 {			h.reclaim(npages) // 回收一部分内存		}		s = h.allocSpan(npages, spanAllocHeap, spanclass) // 申请内存	})	if s != nil {		if needzero && s.needzero != 0 {			memclrNoHeapPointers(unsafe.Pointer(s.base()), s.npages<<_PageShift)		}		s.needzero = 0	}	return s}
```

#### allocSpan

```go
func (h *mheap) allocSpan(npages uintptr, typ spanAllocType, spanclass spanClass) (s *mspan) {
	// Function-global state.
	gp := getg()
	base, scav := uintptr(0), uintptr(0)

	// On some platforms we need to provide physical page aligned stack
	// allocations. Where the page size is less than the physical page
	// size, we already manage to do this by default.
	needPhysPageAlign := physPageAlignedStacks && typ == spanAllocStack && pageSize < physPageSize

	// If the allocation is small enough, try the page cache!
	// The page cache does not support aligned allocations, so we cannot use
	// it if we need to provide a physical page aligned stack allocation.
  
  
 //这里会根据需要分配的内存大小再判断一次：
//如果要分配的页数小于pageCachePages/4=64/4=16页，那么就尝试从pcache申请内存；
//如果申请的内存比较大或者线程的页缓存中内存不足，会通过runtime.pageAlloc.alloc从页堆分配内存；
//如果页堆上内存不足，那么就mheap的grow方法从系统上申请内存，然后再调用pageAlloc的alloc分配内存；
  
	pp := gp.m.p.ptr()
	if !needPhysPageAlign && pp != nil && npages < pageCachePages/4 { //申请的内存比较小,尝试从pcache申请内存
		c := &pp.pcache

		// If the cache is empty, refill it.
		if c.empty() {
			lock(&h.lock)
			*c = h.pages.allocToCache()
			unlock(&h.lock)
		}

		// Try to allocate from the cache.
		base, scav = c.alloc(npages)
		if base != 0 {
			s = h.tryAllocMSpan()
			if s != nil {
				goto HaveSpan
			}
			// We have a base but no mspan, so we need
			// to lock the heap.
		}
	}

	// For one reason or another, we couldn't get the
	// whole job done without the heap lock.
	lock(&h.lock) // 对mheap上锁, 这里的锁是全局锁

	if needPhysPageAlign {
		// Overallocate by a physical page to allow for later alignment.
		npages += physPageSize / pageSize
	}

	if base == 0 {
		// Try to acquire a base address.
		base, scav = h.pages.alloc(npages) // 内存比较大或者线程的页缓存中内存不足，从mheap的pages上获取内存
		if base == 0 { // 内存也不够，那么进行扩容
			if !h.grow(npages) {  // 从操作系统申请
				unlock(&h.lock)
				return nil
			}
			base, scav = h.pages.alloc(npages) // 重新申请内存
			if base == 0 { // 内存不足时抛出异常
				throw("grew heap, but no adequate free space found")
			}
		}
	}
	if s == nil {
		// We failed to get an mspan earlier, so grab
		// one now that we have the heap lock.
		s = h.allocMSpanLocked() // 分配一个mspan对象
	}

	if needPhysPageAlign { // 需要对齐
		allocBase, allocPages := base, npages
		base = alignUp(allocBase, physPageSize)
		npages -= physPageSize / pageSize

		// Return memory around the aligned allocation.
		spaceBefore := base - allocBase
		if spaceBefore > 0 {
			h.pages.free(allocBase, spaceBefore/pageSize)
		}
		spaceAfter := (allocPages-npages)*pageSize - spaceBefore
		if spaceAfter > 0 {
			h.pages.free(base+npages*pageSize, spaceAfter/pageSize)
		}
	}

	unlock(&h.lock)

HaveSpan:
	// At this point, both s != nil and base != 0, and the heap
	// lock is no longer held. Initialize the span.
	s.init(base, npages) // 初始化新的page
	if h.allocNeedsZero(base, npages) {
		s.needzero = 1
	}
	nbytes := npages * pageSize
	if typ.manual() {
		s.manualFreeList = 0
		s.nelems = 0
		s.limit = s.base() + s.npages*pageSize
		s.state.set(mSpanManual)
	} else {
		// We must set span properties before the span is published anywhere
		// since we're not holding the heap lock.
		s.spanclass = spanclass
		if sizeclass := spanclass.sizeclass(); sizeclass == 0 {
			s.elemsize = nbytes
			s.nelems = 1

			s.divShift = 0
			s.divMul = 0
			s.divShift2 = 0
			s.baseMask = 0
		} else {
			s.elemsize = uintptr(class_to_size[sizeclass])
			s.nelems = nbytes / s.elemsize

			m := &class_to_divmagic[sizeclass]
			s.divShift = m.shift
			s.divMul = m.mul
			s.divShift2 = m.shift2
			s.baseMask = m.baseMask
		}

		// Initialize mark and allocation structures.
		s.freeindex = 0
		s.allocCache = ^uint64(0) // all 1s indicating all free.
		s.gcmarkBits = newMarkBits(s.nelems)
		s.allocBits = newAllocBits(s.nelems)

		// It's safe to access h.sweepgen without the heap lock because it's
		// only ever updated with the world stopped and we run on the
		// systemstack which blocks a STW transition.
		atomic.Store(&s.sweepgen, h.sweepgen)

		// Now that the span is filled in, set its state. This
		// is a publication barrier for the other fields in
		// the span. While valid pointers into this span
		// should never be visible until the span is returned,
		// if the garbage collector finds an invalid pointer,
		// access to the span may race with initialization of
		// the span. We resolve this race by atomically
		// setting the state after the span is fully
		// initialized, and atomically checking the state in
		// any situation where a pointer is suspect.
		s.state.set(mSpanInUse)
	}

	// Commit and account for any scavenged memory that the span now owns.
	if scav != 0 {
		// sysUsed all the pages that are actually available
		// in the span since some of them might be scavenged.
		sysUsed(unsafe.Pointer(base), nbytes)
		atomic.Xadd64(&memstats.heap_released, -int64(scav))
	}
	// Update stats.
	if typ == spanAllocHeap {
		atomic.Xadd64(&memstats.heap_inuse, int64(nbytes))
	}
	if typ.manual() {
		// Manually managed memory doesn't count toward heap_sys.
		memstats.heap_sys.add(-int64(nbytes))
	}
	// Update consistent stats.
	stats := memstats.heapStats.acquire()
	atomic.Xaddint64(&stats.committed, int64(scav))
	atomic.Xaddint64(&stats.released, -int64(scav))
	switch typ {
	case spanAllocHeap:
		atomic.Xaddint64(&stats.inHeap, int64(nbytes))
	case spanAllocStack:
		atomic.Xaddint64(&stats.inStacks, int64(nbytes))
	case spanAllocPtrScalarBits:
		atomic.Xaddint64(&stats.inPtrScalarBits, int64(nbytes))
	case spanAllocWorkBuf:
		atomic.Xaddint64(&stats.inWorkBufs, int64(nbytes))
	}
	memstats.heapStats.release()

	// Publish the span in various locations.

	// This is safe to call without the lock held because the slots
	// related to this span will only ever be read or modified by
	// this thread until pointers into the span are published (and
	// we execute a publication barrier at the end of this function
	// before that happens) or pageInUse is updated.
	h.setSpans(s.base(), npages, s)  // // 建立mheap与mspan之间的联系

	if !typ.manual() {
		// Mark in-use span in arena page bitmap.
		//
		// This publishes the span to the page sweeper, so
		// it's imperative that the span be completely initialized
		// prior to this line.
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		atomic.Or8(&arena.pageInUse[pageIdx], pageMask)

		// Update related page sweeper stats.
		atomic.Xadd64(&h.pagesInUse, int64(npages))
	}

	// Make sure the newly allocated span will be observed
	// by the GC before pointers into the span are published.
	publicationBarrier()

	return s
}
```

#### grow-向操作系统申请内存

```go
// Try to add at least npage pages of memory to the heap,
// returning whether it worked.
//
// h.lock must be held.
func (h *mheap) grow(npage uintptr) bool {
	assertLockHeld(&h.lock)

	// We must grow the heap in whole palloc chunks.
	ask := alignUp(npage, pallocChunkPages) * pageSize

	totalGrowth := uintptr(0)
	// This may overflow because ask could be very large
	// and is otherwise unrelated to h.curArena.base.
	end := h.curArena.base + ask
	nBase := alignUp(end, physPageSize)
  // 内存不够则或溢出调用sysAlloc申请内存
	if nBase > h.curArena.end || /* overflow */ end < h.curArena.base {
		// Not enough room in the current arena. Allocate more
		// arena space. This may not be contiguous with the
		// current arena, so we have to request the full ask.
		av, asize := h.sysAlloc(ask)
		if av == nil {
			print("runtime: out of memory: cannot allocate ", ask, "-byte block (", memstats.heap_sys, " in use)\n")
			return false
		}

    // 重新设置curArena的值
		if uintptr(av) == h.curArena.end {
			// The new space is contiguous with the old
			// space, so just extend the current space.
			h.curArena.end = uintptr(av) + asize
		} else {
			// The new space is discontiguous. Track what
			// remains of the current space and switch to
			// the new space. This should be rare.
			if size := h.curArena.end - h.curArena.base; size != 0 {
				h.pages.grow(h.curArena.base, size)
				totalGrowth += size
			}
			// Switch to the new space.
			h.curArena.base = uintptr(av)
			h.curArena.end = uintptr(av) + asize
		}

		// The memory just allocated counts as both released
		// and idle, even though it's not yet backed by spans.
		//
		// The allocation is always aligned to the heap arena
		// size which is always > physPageSize, so its safe to
		// just add directly to heap_released.
		atomic.Xadd64(&memstats.heap_released, int64(asize))
		stats := memstats.heapStats.acquire()
		atomic.Xaddint64(&stats.released, int64(asize))
		memstats.heapStats.release()

		// Recalculate nBase.
		// We know this won't overflow, because sysAlloc returned
		// a valid region starting at h.curArena.base which is at
		// least ask bytes in size.
		nBase = alignUp(h.curArena.base+ask, physPageSize)
	}

	// Grow into the current arena.
	v := h.curArena.base
	h.curArena.base = nBase
	h.pages.grow(v, nBase-v)
	totalGrowth += nBase - v

	// We just caused a heap growth, so scavenge down what will soon be used.
	// By scavenging inline we deal with the failure to allocate out of
	// memory fragments by scavenging the memory fragments that are least
	// likely to be re-used.
	if retained := heapRetained(); retained+uint64(totalGrowth) > h.scavengeGoal {
		todo := totalGrowth
		if overage := uintptr(retained + uint64(totalGrowth) - h.scavengeGoal); todo > overage {
			todo = overage
		}
		h.pages.scavenge(todo, false)
	}
	return true
}
```

#### sysAlloc

grow会通过curArena的end值来判断是不是需要从系统申请内存；如果end小于nBase那么会调用runtime.mheap.sysAlloc方法从操作系统中申请更多的内存 :

```go
// sysAlloc allocates heap arena space for at least n bytes. The
// returned pointer is always heapArenaBytes-aligned and backed by
// h.arenas metadata. The returned size is always a multiple of
// heapArenaBytes. sysAlloc returns nil on failure.
// There is no corresponding free function.
//
// sysAlloc returns a memory region in the Prepared state. This region must
// be transitioned to Ready before use.
//
// h must be locked.
func (h *mheap) sysAlloc(n uintptr) (v unsafe.Pointer, size uintptr) {
	assertLockHeld(&h.lock)

	n = alignUp(n, heapArenaBytes)

	// First, try the arena pre-reservation.
  // 尝试先在预先保留的内存中申请一块可以使用的空间
	v = h.arena.alloc(n, heapArenaBytes, &memstats.heap_sys)
	if v != nil {
		size = n
		goto mapped
	}

	// Try to grow the heap at a hint address.
	for h.arenaHints != nil { // 根据页堆的arenaHints在目标地址上尝试扩容
		hint := h.arenaHints
		p := hint.addr
		if hint.down {
			p -= n
		}
		if p+n < p {
			// We can't use this, so don't ask.
			v = nil
		} else if arenaIndex(p+n-1) >= 1<<arenaBits {
			// Outside addressable heap. Can't use.
			v = nil
		} else {
			v = sysReserve(unsafe.Pointer(p), n) // 从操作系统中申请内存
		}
		if p == uintptr(v) {
			// Success. Update the hint.
			if !hint.down {
				p += n
			}
			hint.addr = p
			size = n
			break
		}
		// Failed. Discard this hint and try the next.
		//
		// TODO: This would be cleaner if sysReserve could be
		// told to only return the requested address. In
		// particular, this is already how Windows behaves, so
		// it would simplify things there.
		if v != nil {
			sysFree(v, n, nil)
		}
		h.arenaHints = hint.next
		h.arenaHintAlloc.free(unsafe.Pointer(hint))
	}

	if size == 0 {
		if raceenabled {
			// The race detector assumes the heap lives in
			// [0x00c000000000, 0x00e000000000), but we
			// just ran out of hints in this region. Give
			// a nice failure.
			throw("too many address space collisions for -race mode")
		}

		// All of the hints failed, so we'll take any
		// (sufficiently aligned) address the kernel will give
		// us.
		v, size = sysReserveAligned(nil, n, heapArenaBytes)
		if v == nil {
			return nil, 0
		}

		// Create new hints for extending this region.
		hint := (*arenaHint)(h.arenaHintAlloc.alloc())
		hint.addr, hint.down = uintptr(v), true
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
		hint = (*arenaHint)(h.arenaHintAlloc.alloc())
		hint.addr = uintptr(v) + size
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
	}

	// Check for bad pointers or pointers we can't use.
	......

	// Transition from Reserved to Prepared.
	sysMap(v, size, &memstats.heap_sys) // 将内存由Reserved转为Prepared

mapped:
	// Create arena metadata.
	// 初始化一个新的heapArena来管理刚刚申请的内存
	for ri := arenaIndex(uintptr(v)); ri <= arenaIndex(uintptr(v)+size-1); ri++ {
		l2 := h.arenas[ri.l1()]
		if l2 == nil {
			// Allocate an L2 arena map.
			l2 = (*[1 << arenaL2Bits]*heapArena)(persistentalloc(unsafe.Sizeof(*l2), sys.PtrSize, nil))
			if l2 == nil {
				throw("out of memory allocating heap arena map")
			}
			atomic.StorepNoWB(unsafe.Pointer(&h.arenas[ri.l1()]), unsafe.Pointer(l2))
		}

		if l2[ri.l2()] != nil {
			throw("arena already initialized")
		}
		var r *heapArena
		r = (*heapArena)(h.heapArenaAlloc.alloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gcMiscSys))
		if r == nil {
			r = (*heapArena)(persistentalloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gcMiscSys))
			if r == nil {
				throw("out of memory allocating heap arena metadata")
			}
		}

		// Add the arena to the arenas list.
		if len(h.allArenas) == cap(h.allArenas) {
			size := 2 * uintptr(cap(h.allArenas)) * sys.PtrSize
			if size == 0 {
				size = physPageSize
			}
			newArray := (*notInHeap)(persistentalloc(size, sys.PtrSize, &memstats.gcMiscSys))
			if newArray == nil {
				throw("out of memory allocating allArenas")
			}
			oldSlice := h.allArenas
			*(*notInHeapSlice)(unsafe.Pointer(&h.allArenas)) = notInHeapSlice{newArray, len(h.allArenas), int(size / sys.PtrSize)}
			copy(h.allArenas, oldSlice)
			// Do not free the old backing array because
			// there may be concurrent readers. Since we
			// double the array each time, this can lead
			// to at most 2x waste.
		}
    
    // // 将创建heapArena放入到arenas列表中
		h.allArenas = h.allArenas[:len(h.allArenas)+1]
		h.allArenas[len(h.allArenas)-1] = ri

		// Store atomically just in case an object from the
		// new heap arena becomes visible before the heap lock
		// is released (which shouldn't happen, but there's
		// little downside to this).
		atomic.StorepNoWB(unsafe.Pointer(&l2[ri.l2()]), unsafe.Pointer(r))
	}

	// Tell the race detector about the new heap memory.
	if raceenabled {
		racemapshadow(v, size)
	}

	return
}
```

#### sysReserve-mmap系统调用

在Linux系统上调用mmap函数

**Mmap**：Mmap是内存映射文件的缩写。它是一种无需调用系统调用就能读写文件的方式。操作系统预留了程序虚拟地址的一块，直接 “映射 “到文件中的一块。因此，如果程序从该部分地址空间读取数据，就会获得驻留在文件相应部分的数据。如果文件的那部分数据恰好驻留在缓冲区缓存中，那么在第一次访问时，只需将映射后的块的虚拟地址映射到相应的缓冲区缓存页的物理地址即可，以后不会再调用系统调用或其他陷阱。如果文件数据不在缓冲区缓存中，访问映射区域会产生一个页面故障，提示内核去从磁盘中获取相应的数据

分配、释放内存在不同的平台 (linux/win/bsd) 有差异，所以 Go 先对这些基础函数进行了封装，如Linux在runtime/mem-linux.go中

```go
func sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer {
	p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
	if err != 0 {
		return nil
	}
	return p
}

// mmap is used to do low-level memory allocation via mmap. Don't allow stack
// splits, since this function (used by sysAlloc) is called in a lot of low-level
// parts of the runtime and callers often assume it won't acquire any locks.
// go:nosplit
func mmap(addr unsafe.Pointer, n uintptr, prot, flags, fd int32, off uint32) (unsafe.Pointer, int) {
	args := struct {
		addr            unsafe.Pointer
		n               uintptr
		prot, flags, fd int32
		off             uint32
		ret1            unsafe.Pointer
		ret2            int
	}{addr, n, prot, flags, fd, off, nil, 0}
	libcCall(unsafe.Pointer(funcPC(mmap_trampoline)), unsafe.Pointer(&args))
	return args.ret1, args.ret2
}
func mmap_trampoline()
```

另外，runtime还提供了sysReserve，sysUnused，sysUsed，sysFree，sysFault，sysMap一系列方法，将内存在不同的下列四种状态间切换

- None:内存没有被保留或者映射，是地址空间的默认状态
- Reserve:运行时持有该地址空间，但是访问该内存会导致错误
- Prepare:内存被保留
- Ready:可以被安全访问

![image](/images/golang-memory4.jpg)

finish

至此，内存申请的整个流程结束。

本文引自[这里](https://www.spider1998.com/2021/12/29/Golang%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)





