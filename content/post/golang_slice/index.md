---
title: Golang 切片Slice源码走读
summary: 本文是对Golang Slice的关键源码进行走读，并记录使用时容易犯的一些错误。示例可能来源于网络、各类书籍，对应的Golang版本是1.21.9，仅供个人学习使用。
date: 2024-05-31

authors:
  - admin

tags:
  - Golang Slice
  - Data Types
---


切片slice是golang中非常经典的数据结构，其定位可以类比其他语言中的动态数组。	切片中的元素存放在一块内存地址连续的区域，使用索引可以快速检索到指定位置的元素；切片长度和容量是可变的，在使用过程中可以根据需要进行扩容。
本文对应的Golang版本是1.21.9


### 数据结构
```golang
// src/runtime/slice.go
type slice struct {
    array unsafe.Pointer
    len int
    cap int
}
```
array 指针指向底层数组， len表示切片长度，cap表示底层数组容量。

### 初始化
声明和初始化主要通过 1) 变量声明 2) 字面量 3) 使用内置函数make() 4) 从切片和数组中切取
```golang
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```
1. 调用 math.MulUintptr 的方法，结合每个元素的大小以及切片的容量，计算出初始化切片所需要的内存空间大小。
2. 如果容量超限，len 取负值或者 len 超过 cap，直接 panic
3. 调用位于 runtime/malloc.go 文件中的 mallocgc 方法，为切片进行内存空间的分配


### 元素追加
```
a := []int{1,2,3,4,5}
b := a[1:4]
b = append(b, 10) // 此时元素a[4]将由5变成0

此时需要借助扩展表达式 (完整切片表达式)
a := []int{1,2,3,4,5}
b := a[1:4:4]
b = append(b, 10)
```

### 切片扩容

```golang
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	oldLen := newLen - num
	...

	if newLen < 0 {
		panic(errorString("growslice: len out of range"))
	}
	if et.Size_ == 0 {
		// 倘若元素大小为 0，则无需分配空间直接返回
		return slice{unsafe.Pointer(&zerobase), newLen, newLen}
	}

	newcap := oldCap  // 计算扩容后数组的容量
	doublecap := newcap + newcap   // 取原容量两倍的容量数值
	if newLen > doublecap {
		// 倘若新的容量大于原容量的两倍，直接取新容量作为数组扩容后的容量
		newcap = newLen
	} else {
		const threshold = 256
		// 倘若原容量小于 256，则扩容后新容量为原容量的两倍
		if oldCap < threshold {
			newcap = doublecap
		} else {
			// 在原容量的基础上，对原容量 * 5/4 
            // 循环执行上述操作，直到扩容后的容量已经大于等于预期的新容量为止
			for 0 < newcap && newcap < newLen {
				// Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			// 倘若数值越界了，则取预期的新容量 cap 封顶
			if newcap <= 0 {
				newcap = newLen
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// 基于容量，确定新数组容器所需要的内存空间大小 capmem
	switch {
	// 倘若数组元素的大小为 1，则新容量大小为 1 * newcap.
    // 同时会针对 span class 进行取整
	case et.Size_ == 1:
		...
	// 倘若数组元素为指针类型，则根据指针占用空间结合元素个数计算空间大小
    // 并会针对 span class 进行取整
	case et.Size_ == goarch.PtrSize:
		...
	// 倘若元素大小为 2 的指数，则直接通过位运算进行空间大小的计算  
	case isPowerOfTwo(et.Size_):
		...
	/ 兜底分支：根据元素大小乘以元素个数
    // 再针对 span class 进行取整  
	default:
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	}
	// 进行实际的切片初始化操作
    var p unsafe.Pointer
    // 非指针类型
    if et.ptrdata == 0 {
        p = mallocgc(capmem, nil, false)
        // ...
    } else {
        // 指针类型
        p = mallocgc(capmem, et, true)
        // ...
    }
    // 将切片的内容拷贝到扩容后的位置 p 
    memmove(p, old.array, lenmem)
	return slice{p, newLen, newcap}
}
```

切片的扩容流程源码位于 runtime/slice.go 文件的 growslice 方法当中，其中核心步骤如下：

• 倘若扩容后预期的新容量小于原切片的容量，则 panic

• 倘若切片元素大小为 0（元素类型为 struct{}），则直接复用一个全局的 zerobase 实例，直接返回

• 倘若预期的新容量超过老容量的两倍，则直接采用预期的新容量

• 倘若老容量小于 256，则直接采用老容量的2倍作为新容量

• 倘若老容量已经大于等于 256，则在老容量的基础上扩容 1/4 的比例并且累加上 192 的数值，持续这样处理，直到得到的新容量已经大于等于预期的新容量为止

• 结合 mallocgc 流程中，对内存分配单元 mspan 的等级制度，推算得到实际需要申请的内存空间大小

• 调用 mallocgc，对新切片进行内存初始化

• 调用 memmove 方法，将老切片中的内容拷贝到新切片中

• 返回扩容后的新切片

```golang
func Test_slice(t *testing.T){
    s := make([]int,512)  
    s = append(s,1)
    t.Logf("len of s: %d, cap of s: %d",len(s),cap(s))
}
len: 513, cap: 848

1. 切片扩容操作：由于切片 s 原有容量为 512，已经超过了阈值 256，因此对其进行扩容操作会采用的计算方式为 512 * (512 + 3*256)/4 = 832
2. 然后结合分配内存的 mallocgc 流程，将 8byte * 832 = 6656 byte 补齐到6784 byte 的这一档次。扩容后实际的新容量为 cap = 6784/8 = 848。

// class  bytes/obj  bytes/span  objects  tail waste  max waste  min align
//     1          8        8192     1024           0     87.50%          8
//     2         16        8192      512           0     43.75%         16
//     3         24        8192      341           8     29.24%          
// ...
//    48       6528       32768        5         128      6.23%        128
//    49       6784       40960        6         256      4.36%        128 
```



### 元素删除

```golang
// 删除尾部元素
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    // [0,1,2,3]
    s = s[0:len(s)-1]
}

// 删除中间元素
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    // 删除 index = 2 的元素
    s = append(s[:2],s[3:]...)
    // s: [0,1,3,4], len: 4, cap: 5
    t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
}

// 删除所有元素
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    s = s[:0]
    // s: [], len: 0, cap: 5
    t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
}
```


### 切片拷贝

```golang

// s 和 s1 的地址是一致的.

func Test_slice(t *testing.T) {
    s := []int{0, 1, 2, 3, 4}
    s1 := s
    t.Logf("address of s: %p, address of s1: %p", s, s1)
}

// s 和 s1 的地址是相互独立的
func Test_slice(t *testing.T) {
    s := []int{0, 1, 2, 3, 4}
    s1 := make([]int, len(s))
    copy(s1, s)
    t.Logf("s: %v, s1: %v", s, s1)
    t.Logf("address of s: %p, address of s1: %p", s, s1)
}

注意: 
1. 关于slice的切割赋值，如果cap比较大，仍然会导致高内存消耗，有内存泄漏的风险。此时，应该使用copy(src, dst)函数进行拷贝。
2. Slice并非并发安全的数据结构，使用时需要注意并发安全的问题。
``` 
