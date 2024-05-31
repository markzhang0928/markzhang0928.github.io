---
title: Golang 切片Map源码走读
summary: 本文是对Golang Map的关键源码进行走读，并记录使用时容易犯的一些错误。示例可能来源于网络、各类书籍，对应的Golang版本是1.21.9，仅供个人学习使用。
date: 2024-05-31

authors:
  - admin

tags:
  - Golang Map
  - Data Types
---


## 1. 数据结构

map 又称字典，是一种常用的数据结构，核心特征包含下述三点： 
- 1）存储基于 key-value 对映射的模式； 
- 2）基于 key 维度实现存储数据的去重； 
- 3）读、写、删操作控制，时间复杂度 O(1).

可以通过make 或字面量 两种方式进行初始化。使用make(map[string]int, xxx) 指定容量大小进行初始化，不必动态的创建桶并重新分配桶中的元素，
相比另一种方法更加高效。

```go
type hmap struct {
    count           int              // 元素个数，调用len(map)时，直接返回此值
    flags           uint8            // map 状态标识，可以标识出 map 是否被 goroutine 并发读写；
    B               uint8            // buckets 数组的大小的对数 log_2
    noverflow       uint             // map 中溢出桶的数量；
    hash0           uint32           // 计算key的哈希时会传入哈希函数
    buckets         unsafe.Pointer   // 指向buckets数组，大小为2^B; 如果元素个数为0， 就为nil
    oldbuckets      unsafe.Pointer   // 扩容时，bucket长度时oldbuckets的两倍
    nevacuate       uintptr          // 指示扩容进度，小于此地址的buckets完成迁移
    extra           *mapextra       // 预申请的溢出桶.
}

// bmap 就是 map 中的桶，可以存储 8 组 key-value 对的数据，以及一个指向下一个溢出桶的指针；
type bmap struct {
	tophash [8]uint8 // 存储key的哈希值的高八位
	data    []byte  //  存储key-value对: key/key/key/.../value/value/value/...
	overflow *bmap  // 溢出bucket的地址
}
```

## 2. map的初始化过程


### makemap
```golang
func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 检查bucket个数乘以bucket大小，看是否超出了内存申请的限制
	mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
	if overflow || mem > maxAlloc {
		// 如果超限，会将 hint 置为零；
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// 找到一个B，使得map的装载因子在正常范围内，计算桶数组的容量 B；
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

    // 初始化hash table；如果B等于0，那么buckets就会在赋值的时候再分配。（lazy模式）
	// 如果长度比较大，清理内存花费的时间就会长一些。
	if h.B != 0 {
		var nextOverflow *bmap
		// 调用 makeBucketArray 方法，初始化桶数组 hmap.buckets；
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			// 倘若 map 容量较大，会提前申请一批溢出桶 hmap.extra.
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```


### 负载因子
hash map使用负载因子来衡量hash表的冲突情况， 负载因子 =  键数量/ bucket数量; 当负载因子大于某个阈值时，需要rehash。
(redis 的map实现负载因子大于1就会触发rehash；golang的map实现负载因子大于6.5就会触发扩容)

```golang
const (
    loadFactorNum = 13
    loadFactorDen = 2
    goarch.PtrSize = 8
    bucketCnt = 8 // abi.MapBucketCount
)

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
    return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

func bucketShift(b uint8) uintptr {
    return uintptr(1) << (b & (goarch.PtrSize*8 - 1))
}

```

### makeBucketArray
makeBucketArray 方法会进行桶数组的初始化，并根据桶的数量决定是否需要提前作溢出桶的初始化.

```golang
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	//  map 的桶数组申请内存，在桶数组的指数 b >= 4时（桶数组的容量 >= 52 ），会需要提前创建溢出桶.
	if b >= 4 {
		// 通过 base 记录桶数组的长度，不包含溢出桶；通过 nbuckets 记录累加上溢出桶后，桶数组的总长度.
		nbuckets += bucketShift(b - 4)
		sz := t.Bucket.Size_ * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.Bucket.Size_
		}
	}
    // 如果 dirtyalloc 为空，将分配一个新的后备数组。
	// 否则 dirtyalloc 将被清除并作为后备数组重新使用。
	if dirtyalloc == nil {
		// 调用 newarray 方法为桶数组申请内存空间，连带着需要初始化的溢出桶：
		buckets = newarray(t.Bucket, int(nbuckets))
	} else {
		buckets = dirtyalloc
		size := t.Bucket.Size_ * nbuckets
		if t.Bucket.PtrBytes != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// 倘若 base != nbuckets，说明需要创建溢出桶，会基于地址偏移的方式，通过 nextOverflow 指向首个溢出桶的地址.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
		// 倘若需要创建溢出桶，会在将最后一个溢出桶的 overflow 指针指向 buckets 数组，以此来标识申请的溢出桶已经用完.
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```



### 3. key定位过程
key经过哈希计算后得到哈希值，根据B个bit位得到桶的数量。当两个不同的key落在同一个桶中，冲突解决手段是用链表法：
(TODO)


### 4. map的赋值过程
(TODO)


### 5. map的删除过程
(TODO)

### 6. map的扩容过程
(TODO)

### 7. map的遍历过程
(TODO)


### 使用注意事项:
1. golang map的大小无法收缩的问题？
- 背景: 它的大小不会自动收缩，也就是说，即使你从map中删除了一些元素，它占用的内存空间并不会自动减小。这个特性是由Go的内存管理策略决定的。当你向map中添加元素时，如果当前的存储空间不足以存储新的元素， 
Go会自动为map分配更大的内存空间。然而，当你删除元素时，Go并不会自动减小map的存储空间。这是因为频繁的内存分配和释放是非常昂贵的操作，会大大影响性能。

- 解决方案：
如果你的应用中有一个非常大的map，并且在使用过程中会有大量的元素被删除，你可能需要手动管理map的大小。这可以通过以下方式做到：
  - 1. 创建一个新的map，将旧map中剩余的元素复制到新map中，然后丢弃旧的map。由于新的map只包含需要的元素，所以它的大小会比旧的map小。(缺陷: 在复制之后，知道下一次垃圾回收之前，旧map占用的内存空间不会被释放)。
  - 2. 更改map的类型，比如: map[int]*[128]byte.
   | 步骤 | map[int][128]byte | map[int]*[128]byte |  
   | --- | --- | --- |
   | 分配一个空map | 0MB | 0MB |  
   | 添加100万个元素 | 461MB | 182MB |
   | 删除100万个元素并执行GC | 293MB | 38MB |
  - 3. 如果你的map的大小会频繁变化，你可能需要使用一种更复杂的数据结构，如使用sync.Pool来缓存和复用map，或者使用第三方库提供的并发安全且可以动态收缩的map。
  - 需要注意的是，以上的解决方案都会带来额外的开销，所以只有在确实需要管理map的大小时才应该使用。在大多数情况下，Go的默认行为已经足够好了。

2. 如何比较两个map是否相等? 遍历，或者通过reflect.DeepEqual。
