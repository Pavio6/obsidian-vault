## Golang map底层原理

map又称字典，底层结构基于hash和桶数组
1. 存储基于key-value对映射的模式
2. 基于key维度实现存储数据的去重
3. 读、写、删操作控制，时间复杂度 O(1)

**初始化**
```go

myMap1 := make(map[int]int,2) // 使用make初始化，并指定map预分配的容量
myMap2 := make(map[int]int) // 不显示声明容量，默认容量为0
myMap3 := map[int]int{ // 初始化并赋值
	1:2,
	3:4,
}
```

map中，key的数据结构要是可比较的，slice、map、func 不可比较
##### 读map
直接读，倘若key存在，则获取到对应的value值，倘若key不存在或map未初始化，返回val类型的零值作为兜底，但无法区分是val等于0，还是key不存在，可以使用下面方法来解决。
```go
v1 := myMap[10]
v2, ok := myMap[10] // ok是一个bool类型的flag，标识是否成功
```
##### 写map
```go
// 写的操作语法上，必须要求map是初始化过的
myMap[5] = 6
```
##### 删除map中的某个key
```go
delete(myMap, 5) // 5 代表key
```
##### 遍历map
```go
for k,v := range myMap { // 基于k,v依次承接map中的key-value对
	// ...
}

for k := range myMap { // 只关心key 不关注value的值
	// ...
}

```
在执行map的遍历操作时，获取的key-value并没有一个固定的顺序，可能两次遍历的顺序存在差异

**并发冲突** map不是并发安全的数据结构，倘若存在并发读写行为，会抛出fatal error
规则是：
1. 并发读没有问题
2. 并发读写中的"写"是广义的，包含写、更新、删除等操作
3. 读的时候发现其他goroutine在并发写，报错
4. 写的时候发现其他goroutine在并发写，报错

#### hash
- hash的可重入性：相同的key，必然产生相同的hash值
- hash的离散性：只有两个key不相同，不论其相似度的高低，产生的hash值会在整个输出域内均匀地离散化
- hash的单向性：企图通过hash值反向映射回key是无迹可寻的
- hash冲突：由于输入域(key)无穷大，输出域(hash值)有限，因此必然存在不同key映射到相同hash值的情况，称之为hash冲突

#### 桶数组 
map中，会通过长度为2的整数次幂的桶数组进行key-value对的存储：
- 每个桶固定存放8个key-value对
- 倘若超过8个key-value对打到桶数组的同一个索引当中，此时会通过创建桶链表的方式来解决该问题

#### 解决hash冲突
1. 链地址法：当不同的 key 经过哈希计算后落入同一个桶（bucket）时，用溢出桶链表将这些冲突的键值对串联存储。
2. 开发寻址法：当发生hash冲突时，按照某种探测策略查找下一个可用的位置。即所有数据都存储在hash表的数组内。
---

go采用数组+链表（溢出桶）的方法，每个桶（bucket）存储 8 个键值对（优化内存局部性）。
如果桶满了，会链接一个溢出桶（overflow bucket），形成链表结构。
查找时，先定位桶，再遍历链表（最坏情况 O(n)，但概率很低）。

#### map的扩容机制
map扩容机制核心点：
 1. 扩容分为增量扩容（容量翻倍）和等量扩容（重新整理扩容）
 2. 当桶内的key-value总数 / 桶数组的长度 > 6.5 时，发生增量扩容，桶数组长度增长为原值的两倍。
 3. 当桶内溢出桶数量 >= 2^B时（B为桶数组长度的指数，最大为15），发生等量扩容，桶的长度保持为原值。
 4. 采用渐进式扩容，当桶被实际操作时，由使用者负责完成数据迁移，避免因为一次性的全量数据迁移引发性能抖动。

#### map类定义的源码
```go
// hmap - 主控制结构
type hmap struct {
	// 活跃单元数量 == map 大小。必须位于第一个（由 len() 内置函数使用）
    count     int       
    flags     uint8     // 状态标记，用于并发控制和扩容状态
    B         uint8     // 桶数组的大小为 2^B
    noverflow uint16    // 溢出桶的近似数量
    hash0     uint32    // 哈希种子，用于计算键的哈希值
    // 当前使用的桶数组指针，包含 2^B 个 bucket 的数组
    // 如果 count==0，则可能为 nil
    buckets    unsafe.Pointer 
    // 扩容时的旧桶数组指针，前一个 bucket 数组，大小为之前的 bucket 的一半，
    // 仅在增长时为非零值
    oldbuckets unsafe.Pointer 
    nevacuate  uintptr        // 扩容进度标记
    
    extra *mapextra // 可选字段，用于存储溢出桶相关数据
}
```

#### mapextra - 溢出桶管理
```go
// mapextra 溢出桶结构体
type mapextra struct {
    
    // overflow指向当前桶数组（hmap.buckets）的溢出桶
    overflow    *[]*bmap
    // oldoverflow指向扩容前旧桶数组（hmap.oldbuckets）的溢出桶
    oldoverflow *[]*bmap
    // nextOverflow指向一个预分配的空闲溢出桶，用于快速分配新的溢出桶，
    // 避免频繁的内存分配操作，提高性能
    nextOverflow *bmap
}
```

#### bmap - 桶结构
```go
const (
	MapBucketCountBits = 3
	MapBucketCount = 1 << MapBucketCountBits // 8
)  

type bmap struct {
	// 存储键哈希值的高 8 位，用于快速匹配
	tophash [abi.MapBucketCount]uint8

	// 实际的键值对存储在 tophash 之后（由编译器动态生成）
    // keys [8]keytype
    // values [8]valuetype
    
    // 溢出桶指针，当桶满时链接到下一个桶
    // overflow *bmap
	
}
```

#### 构造方法
```go
// makemap 创建一个新的 map 实例
// 参数:
//   - t: map 的类型信息
//   - hint: 预期的键值对数量，用于预分配内存
//   - h: 可选的已存在的 hmap 指针（用于 map 字面量优化）
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 计算所需内存，防止溢出
    mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
    if overflow || mem > maxAlloc {
        hint = 0 // 溢出时重置为默认大小
    }

    // 初始化 hmap 结构体
    if h == nil {
        h = new(hmap)
    }
    h.hash0 = uint32(rand()) // 生成随机哈希种子，防止哈希碰撞攻击

    // 根据 hint 计算合适的桶数量 (2^B)
    // 当预期元素数量超过桶数量的负载因子阈值时，增加 B
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 分配初始哈希表
    // 若 B=0，则延迟分配 buckets（在首次赋值时）
    // 若 hint 很大，初始化内存可能耗时较长
    if h.B != 0 {
        var nextOverflow *bmap
        h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
        if nextOverflow != nil {
            h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow // 保存预分配的溢出桶
        }
    }

    return h
}

// 倘若map预分配容量小于等于8，B取0，桶的个数为1
// 保证map预分配容量小于等于桶数组长度*6.5
func overLoadFactor(count int, B uint8) bool {
	return count > abi.MapBucketCount && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

#### 读流程
``` go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 1. 竞态检测和内存检查
	if raceenabled && h != nil {
		callerpc := getcallerpc()
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
	// 2. 空map检查 
	
	if h == nil || h.count == 0 { // map为空或数量为0
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0]) // 返回zeroVal数组第一个元素的地址，也就是对应元素的零值
	}
	// 3. 并发写检查
	if h.flags&hashWriting != 0 { // 是否有并发写操作
		fatal("concurrent map read and map write")
	}
	// 4. 计算hash值，并定位bucket
	hash := t.Hasher(key, uintptr(h.hash0)) // 计算key的hash值，hash0是种子值
	m := bucketMask(h.B) // 计算bucket的掩码 也就是 2^B - 1，按位运算比取模性能更快
	// 定位到初始bucket，将hash与m进行按位与，得到桶索引范围为 [0,2^B-1)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize))) 

	// 5. 处理扩容迁移中的情况
	if c := h.oldbuckets; c != nil { // 正在扩容
		if !h.sameSizeGrow() { // 检查是否为增量扩容
			m >>= 1 // 使用旧的bucket数量
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize))) // 定位到旧bucket
		if !evacuated(oldb) { // 如果旧bucket未迁移
			b = oldb // 从旧bucket中查找
		}
	}
	// 6. 在bucket链中查找key
	top := tophash(hash) // 获取hash值的高八位
bucketloop: // bucketloop是一个标签，标识外层循环，以便在内层循环中直接通过break跳出循环
	for ; b != nil; b = b.overflow(t) {
		// uintptr(0)：表示空指针地址
		for i := uintptr(0); i < abi.MapBucketCount; i++ {
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
				e := add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				if t.IndirectElem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	// 7. 未找到，返回零值
	return unsafe.Pointer(&zeroVal[0])
}

/**
 * 获取当前桶的溢出桶
 * b *bmap - 当前桶的指针
 * t *maptype - 哈希表的类型信息
 * *bmap - 指向当前桶的溢出桶的指针
 */ 
func (b *bmap) overflow(t *maptype) *bmap {
	// 
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.BucketSize)-goarch.PtrSize))
}
```

#### 写流程
```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.Key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.Key.Size_)
	}
	if asanenabled {
		asanread(key, t.Key.Size_)
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}
	hash := t.Hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write.
	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.Bucket) // newarray(t.Bucket, 1)
	}

again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < abi.MapBucketCount; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
					elem = add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.Key.Equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.NeedKeyUpdate() {
				typedmemmove(t.Key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// The current bucket and all the overflow buckets connected to it are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, abi.MapBucketCount*uintptr(t.KeySize))
	}

	// store new key/elem at insert position
	if t.IndirectKey() {
		kmem := newobject(t.Key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.IndirectElem() {
		vmem := newobject(t.Elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.Key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.IndirectElem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

#### 删除流程
```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
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
		return
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}

	hash := t.Hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write (delete).
	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < abi.MapBucketCount; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			k2 := k
			if t.IndirectKey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.Key.Equal(key, k2) {
				continue
			}
			// Only clear key if there are pointers in it.
			if t.IndirectKey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.Key.Pointers() {
				memclrHasPointers(k, t.Key.Size_)
			}
			e := add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			if t.IndirectElem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.Elem.Pointers() {
				memclrHasPointers(e, t.Elem.Size_)
			} else {
				memclrNoHeapPointers(e, t.Elem.Size_)
			}
			b.tophash[i] = emptyOne
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// It would be nice to make this a separate function, but
			// for loops are not currently inlineable.
			if i == abi.MapBucketCount-1 {
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
					i = abi.MapBucketCount - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			// Reset the hash seed to make it more difficult for attackers to
			// repeatedly trigger hash collisions. See issue 25237.
			if h.count == 0 {
				h.hash0 = uint32(rand())
			}
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
}

```

#### 遍历流程
```go
// 数据结构
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/compile/internal/walk/range.go).
	elem        unsafe.Pointer // Must be in second position (see cmd/compile/internal/walk/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}


func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, abi.FuncPCABIInternal(mapiterinit))
	}

	it.t = t
	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/goarch.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/reflectdata/reflect.go
	}
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if !t.Bucket.Pointers() {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	r := uintptr(rand())
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (abi.MapBucketCount - 1))

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
