> [!IMPORTANT]
> 在撰写本文时Golang的最新版本是1.24，请注意部分内容的时效性哦～

听闻Golang在最新的1.24版本中，新增了基于 SwissTable 设计的全新内置 map 类型的实现，没想到技术迭代就发生在身边。那Golang为什么要选择更改map的实现呢，是不是SwissTable性能更好更优秀呢，要想知道根本原因，肯定需要了解两种map底层实现，那我们先从原本的map开始吧！

## 引子

- 源码的位置：`src/map/map_noswiss.go`

1. map的底层结构是什么样子的，容量是固定的吗还是可扩展的？
2. 为什么说map不是并发安全的结构？

## map的结构

### map header

hmap是map的具体实现结构，核心的部分已经高亮出：

```go{11,12,16}
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	clearSeq   uint64

	extra *mapextra // optional fields
}
```

1. `buckets`: 一个数组，每个元素是一个哈希桶，用来存储数据
2. `oldbuckets`: 同样也是哈希桶数组，但是只会在渐进扩容时起作用，是相对旧的桶
3. `extra`: 用来存放溢出的桶指针

```mermaid
---
title: hmap, bmap, extra的关系
---
graph TD
  H[hmap]
  B0[bmap - bucket 0]
  B1[bmap - bucket 1]
  OB0[bmap - overflow 0]
  OB1[bmap - overflow 1]
  OB2[bmap - overflow 2]
  E[mapextra]

  %% 指针引用关系
  H --> B0
  H --> B1
  B0 --> OB0
  OB0 --> OB1
  B1 --> OB2
  E --> OB0
  E --> OB2
  H --> E
```

### map bucket

bmap是map中每个bucket的具体实现结构，但是实际上比表面上看上去更复杂

```go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [abi.OldMapBucketCount]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

- `tophash`：是一个数组，存储了这个bucket中所有key的hash的高八位，容量是一个常量，定义在abi包下，其中也定义了其他相关的变量，为了性能调优。
- `key-value`：并没有显式出现在结构体中，而是手操内存打包再放入结构体中，而且在其中的存储是 key1/key2/value1/value2 的顺序，因为key和key的类型一致，val和val的类型一致，放到一起有利于内存对齐，减少不必要的空间浪费。
- `overflow bucket pointer`: 如果bucket的空间不足，会分配溢出桶来承接新的key-value。

```mermaid
---
title: bucket 和 overflow bucket 的内存结构
---
block-beta
columns 8

block:bucket0:2
  columns 4
  th0_0["tophash: A"] th0_1["B"] th0_2["C"] th0_3["D"] th0_4["E"] th0_5["F"] th0_6["G"] th0_7["H"]
  k0_0["key: k0"] k0_1["k1"] k0_2["k2"] k0_3["k3"] k0_4["k4"] k0_5["k5"] k0_6["k6"] k0_7["k7"]
  v0_0["val: v0"] v0_1["v1"] v0_2["v2"] v0_3["v3"] v0_4["v4"] v0_5["v5"] v0_6["v6"] v0_7["v7"]
  o0["overflow"]
end

block:bucket1:2
  columns 4
  th1_0["tophash: I"] th1_1["J"] th1_2["--"] th1_3["--"] th1_4["--"] th1_5["--"] th1_6["--"] th1_7["--"]
  k1_0["key: k8"] k1_1["k9"] k1_2["--"] k1_3["--"] k1_4["--"] k1_5["--"] k1_6["--"] k1_7["--"]
  v1_0["val: v8"] v1_1["v9"] v1_2["--"] v1_3["--"] v1_4["--"] v1_5["--"] v1_6["--"] v1_7["--"]
  space
end

o0 --> bucket1
```

## extra 的作用

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```

Golang的GC会扫描所有的变量，如果这个变量所在的空间内有地址，就说明存在引用关系，于是下一步就会扫描被引用的地址的变量，直至到达叶节点。问题在于，既然我们的bucket中也是有指针指向下一个overflow bucket的，那么为什么还要一个额外的地址切片记录指针的位置呢？

这个比较复杂，首先是GC在扫描一个变量的时候会先查看变量的Type，其中会涉及到这个变量是否有指针，如果没有指针就跳过，而map是内部实现的，在这方面就比较灵活，首先如果key和value都不含指针就会把map标记为值类型，GC就会跳过，而当map有溢出时，为了防止overflow桶被GC回收，这个时候会标记为指针类型。

- 源码地址 `src/internal/abi/type.go`

```go
type Type struct {
	// ...
	PtrBytes    uintptr // number of (prefix) bytes in the type that can contain pointers
	GCData    *byte
	// ...
}
```

我也没有深入研究，但是在这个Type的结构体中这两个字段是和GC有关的，都是一定程度上影响GC扫描的区域，大家有兴趣也可以去看看。

因此，如果设计上允许，在项目中最好使用非指针类型的key和value，可以有效缓解GC压力。

## make 初始化

### makemap

我们在业务开发中使用map前都会先进行初始化，也就是 `make(map[k]v, hint)`，需要传入这个map的kv的类型，以及map初始容量的大小也就是hint，逻辑所对应的就是下面这段代码：

```go:line-numbers
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
//
// makemap should be an internal detail,
// but widely used packages access it using linkname.
// Notable members of the hall of shame include:
//   - github.com/ugorji/go/codec
//
// Do not remove or change the type signature.
// See go.dev/issue/67401.
//
//go:linkname makemap
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = uint32(rand())

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```

1. 第一步是判断需要分配的内存空间，也就是`hint` `*` `map中每个bucket的大小`，如果预先分配所需的空间太大了，说明`hint`不合理，后面就不会理睬这个值了，而是从最小容量开始渐进扩容。
2. 向golang的内存分配器申请一块hmap的空间。
3. 寻找到一个合适的桶数量，足够容纳hint数量的元素，同时不超过溢出阈值 *13/16*。

```go
// Maximum average load of a bucket that triggers growth is bucketCnt*13/16 (about 80% full)
// Because of minimum alignment rules, bucketCnt is known to be at least 8.
// Represent as loadFactorNum/loadFactorDen, to allow integer math.
loadFactorDen = 2
loadFactorNum = loadFactorDen * abi.OldMapBucketCount * 13 / 16.
```

4. 根据前面得到的B的值，向golang的内存分配器申请所有bucket的空间。需要注意如果hint值是0的话，B也是0，就不会再额外分配内存了；此外当判断不需要overflowbucket的时候也不会为extra分配空间，这是map的懒加载策略，等后面元素增加后再考虑分配内存。

### makeBucketArray

```go:line-numbers
// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.Bucket.Size_ * nbuckets
		up := roundupsize(sz, !t.Bucket.Pointers())
		if up != sz {
			nbuckets = up / t.Bucket.Size_
		}
	}

	if dirtyalloc == nil {
		buckets = newarray(t.Bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.Bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.Bucket.Size_ * nbuckets
		if t.Bucket.Pointers() {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

1. 因为没有初始化容量的map，也就是hint为0时，走渐进式扩容，不走这里的预先分配，因为预先设定了最大元素数量并且桶的数量很少，所以在运行时有可能溢出，但是溢出不太可能。
2. 按照比例 1/16 计算预分配的overflow桶的数量。
3. 调用分配数组的方法为所有的bucket分配空间，包括普通bucket和预分配的overflowbucket，并在结构体中声明好这些分配的overflowbucket空间。
4. 如果传进来了dirtyalloc，则是从clear map走进来的，而不是make map，这个流程中会回收旧的bucket的空间并请求新的空间。感觉这里是因为循环遍历再初始化时间复杂度太高，加上原本可能进行过扩容导致超出了hint，所以没有复用之前的空间，干脆一次性回收。

## access 访问

访问map的元素会执行access的逻辑，比如`val := map[key]`。顺便一提，除了 `mapaccess1` 还有其他的相关函数，主要逻辑都是相同的，只是为了方便使用多了一些额外返回值，比如:

- `mapaccess2`: 多了key是否存在的返回值，`val, exist := map[key]`。
- `mapaccessK`: 多了key的返回值，在range map的时候会用到。

```go:line-numbers=410
// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
// it will return a reference to the zero object for the elem type if
// the key is not in the map.
// NOTE: The returned pointer may keep the whole map live, so don't
// hold onto it for very long.
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := sys.GetCallerPC()
		pc := abi.FuncPCABIInternal(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.Key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.Key.Size_)
	}
	if asanenabled && h != nil {
		asanread(key, t.Key.Size_)
	}
	if h == nil || h.count == 0 {
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map read and map write")
	}
	// ...
}
```

1. 前几行的代码是一些编译选项可以不用理会。
2. 判断有没有初始化，如果没有会发生panic。
3. 这里是读流程，如果发现有goroutine正在写map，会fatal，因为这个结构并不是一个并发安全的结构。

### key 的查找

```go:line-numbers=435
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ...
	hash := t.Hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.Key.Equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				if t.IndirectElem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

1. 使用哈希函数对key进行哈希运算，然后取余，得到的值必定小于普通bucket的数量，根据桶的大小计算偏移量，就可以找到对应的普通bucket。
2. 判断map是否正在扩容以及这个bucket是否已经被迁移，如果尚未迁移就代表数据仍在旧的bucket中，后面在扩容篇细讲。
3. 从普通bucket开始查找key，如果没有那么就去查找overflow bucket，以此递归，最后找到key就返回value，如果没有找到就构建一个空的变量返回。具体每一个bucket的查找流程是先遍历所有的tophash，如果tophash相同意味着hash有可能相同，再去真正的key存储的位置进一步判断。如果tophash不相同，意味着hash一定不同，就不用再去比较key了。之所以这里存储的是tophash而不是hash，是为了提高遍历和对比的速度，毕竟就算去比较完整的hash，也可能不是同一个key，如果直接跳过hash直接对比key，又会面临大key的对比问题，明显消耗更大。

## assign 插入或更新

当我们使用 `map[key] = value`时，会先查找这个key并更新其value，如果当前的map中没有这个key就执行插入的逻辑，先后顺序是很重要的，毕竟如果一个key对应着两个value的话那可就全乱套了。因为代码的重复度很高，这里接不列出了。

1. 首先也是一些校验以及编译选项，包括是否正在并发操作的检验，以及懒加载的机制。
2. 之后还是和 access 查找 key 时相同，找到桶的位置然后对比tophash等等，不同之处在于这里会有扩容相关的机制，同时会在查找时顺便找到一个空位置，方便之后key不存在时直接插入，如果当前bucket和后面挂的所有bucket都满了就分配一个新的 overflow bucket 再插入。
3. 其实可以看到在查询是否并发操作的时候并没有使用原子操作，理论上也会有并发读写或者并发写的同时，程序并没有检测到，所以还是要在代码设计时就去避免～

## delete 删除

1. 主体的逻辑与插入基本相同。不同之处在于会判断删除元素是否是最后一个元素，如果是的话就会在tophash做一个标记，方便在后续执行中判断是否是整个bucket和overflowbucket链条的尾部，从而避免再去遍历剩余的slot和bucket。

```go
// If the bucket now ends in a bunch of emptyOne states,
// change those to emptyRest states.
// It would be nice to make this a separate function, but
// for loops are not currently inlineable.
if i == abi.OldMapBucketCount-1 {
    if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
        goto notLast
    }
} else {
    if b.tophash[i+1] != emptyRest {
        goto notLast
    }
}
for {
    b.tophash[i] = emptyRest
    if i == 0 {
        if b == bOrig {
            break // beginning of initial bucket, we're done.
        }
        // Find previous bucket, continue at its last entry.
        c := b
        for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
        }
        i = abi.OldMapBucketCount - 1
    } else {
        i--
    }
    if b.tophash[i] != emptyOne {
        break
    }
}
```

- tophash的一些预留位置

```go
const (
	// Possible tophash values. We reserve a few possibilities for special marks.
	// Each bucket (including its overflow buckets, if any) will have either all or none of its
	// entries in the evacuated* states (except during the evacuate() method, which only happens
	// during map writes and thus no one else can observe the map during that time).
	emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
	emptyOne       = 1 // this cell is empty
	evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
	minTopHash     = 5 // minimum tophash for a normal filled cell.
)
```

2. 在删除map元素致使map空了的时候，会重新创建一个哈希随机值，可以一定程度上预防有人想要碰撞哈希值。

## grow 渐进扩容

## iterator 遍历

### mapiterinit 初始化遍历结构体


1. **一些编译选项和特殊情况校验**
```go
// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := sys.GetCallerPC()
		racereadpc(unsafe.Pointer(h), callerpc, abi.FuncPCABIInternal(mapiterinit))
	}

	it.t = t
	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{}) != 8+12*goarch.PtrSize {
		throw("hash_iter size incorrect") // see cmd/compile/internal/reflectdata/reflect.go
	}
	it.h = h
	it.clearSeq = h.clearSeq

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
```

2. **保有overflow指针**

接着是一个骚操作，保有了map的overflow指针和oldoverflow指针防止被GC。虽然map是单线程的数据结构，不会并发去操作它，但是如果我们在 range 的时候进行写操作，仍可能导致扩容或者其他竞态发生，而it可以在extra被清空后仍保有引用关系，extra守护不了的，我们来守护。但是在 range 的时候进行写操作会导致不可预料的情况发生，并不是一个稳定可预测的行为，所以最好还是不要这么用。

```go
	if !t.Bucket.Pointers() {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}
```

3. **剩余部分**
随机了一个bucket作为迭代的起始bucket

```go
	// decide where to start
	r := uintptr(rand())
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (abi.OldMapBucketCount - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}
```

```go
for k, v := range m {
    if k == "a" {
        m["newkey"] = 3
    }
}
```

### mapiternext 开始遍历

1. **前置检查**

```go
func mapiternext(it *hiter) {
    h := it.h
    if raceenabled {
        callerpc := sys.GetCallerPC()
        racereadpc(unsafe.Pointer(h), callerpc, abi.FuncPCABIInternal(mapiternext))
    }
    if h.flags&hashWriting != 0 {
        fatal("concurrent map iteration and map write")
    }
    t := it.t
    bucket := it.bucket
    b := it.bptr
    i := it.i
    checkBucket := it.checkBucket
```

* 取出迭代器和 map 的相关字段。
* 检查是否有其他协程在执行写操作，防止出现并发安全问题。

2. **主循环入口**

```go
next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
```

* 如果当前 bucket 为空，说明要进入下一个 bucket。
* 如果已经回到起始 bucket，说明遍历结束，函数返回。
* 如果map正在扩容并且当前bucket还没迁移，就需要额外遍历 oldbuckets。

3. **遍历bucket的每个槽位**

```go
for ; i < abi.OldMapBucketCount; i++ {
    offi := (i + it.offset) & (abi.OldMapBucketCount - 1)
    if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
        // TODO: emptyRest is hard to use here, as we start iterating
        // in the middle of a bucket. It's feasible, just tricky.
        continue
    }
    k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.KeySize))
    if t.IndirectKey() {
        k = *((*unsafe.Pointer)(k))
    }
    e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+uintptr(offi)*uintptr(t.ValueSize))
```

* 遍历当前 bucket 的每个槽位，跳过空槽和已迁移的槽，获得对应的 key 和 element。
* 通过 offset 打乱遍历顺序，也就是可能从8个slot中的任意位置开始遍历。

4. **扩容时额外检查**

```go
if checkBucket != noCheck && !h.sameSizeGrow() {
    // Special case: iterator was started during a grow to a larger size
    // and the grow is not done yet. We're working on a bucket whose
    // oldbucket has not been evacuated yet. Or at least, it wasn't
    // evacuated when we started the bucket. So we're iterating
    // through the oldbucket, skipping any keys that will go
    // to the other new bucket (each oldbucket expands to two
    // buckets during a grow).
    if t.ReflexiveKey() || t.Key.Equal(k, k) {
        // If the item in the oldbucket is not destined for
        // the current new bucket in the iteration, skip it.
        hash := t.Hasher(k, uintptr(h.hash0))
        if hash&bucketMask(it.B) != checkBucket {
            continue
        }
    } else {
        // Hash isn't repeatable if k != k (NaNs).  We need a
        // repeatable and randomish choice of which direction
        // to send NaNs during evacuation. We'll use the low
        // bit of tophash to decide which way NaNs go.
        // NOTE: this case is why we need two evacuate tophash
        // values, evacuatedX and evacuatedY, that differ in
        // their low bit.
        if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
            continue
        }
    }
}
```

- 在map增量扩容时，因为oldbucket中的元素会重新分配到两个bucket中去，一个还是当前的bucket，而另外一个是新的bucket，因此我们在遍历时要跳过将分配到新bucket中的元素，防止它返回两次。
- NaN是一个非常奇怪的数值，有很多特性，我也不是很了解。因为NaN != NaN，所以也没办法通过key的对比去查找到NaN的位置，但是我们只需要知道这个NaN是否会迁移到新的桶，然后是否返回即可。

5. **返回key value给迭代器**

```go
    if it.clearSeq == h.clearSeq &&
        ((b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
            !(t.ReflexiveKey() || t.Key.Equal(k, k))) {
        // This is the golden data, we can return it.
        // OR
        // key!=key, so the entry can't be deleted or updated, so we can just return it.
        it.key = k
        if t.IndirectElem() {
            e = *((*unsafe.Pointer)(e))
        }
        it.elem = e
    } else {
        // The hash table has grown since the iterator was started.
        // The golden data for this key is now somewhere else.
        // Check the current hash table for the data.
        // NOTE: we need to regrab the key as it has potentially been
        // updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
        rk, re := mapaccessK(t, h, k)
        if rk == nil {
            continue // key has been deleted
        }
        it.key = rk
        it.elem = re
    }
    it.bucket = bucket
    if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
        it.bptr = b
    }
    it.i = i + 1
    it.checkBucket = checkBucket
    return
}
```
