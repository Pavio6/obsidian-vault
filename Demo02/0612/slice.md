## 切片（slice）

**切片中的元素存放在一块内存地址连续的区域；切片的长度和容量是可变的，使用过程中可以扩容**

#### 数据结构 Slice Header
``` go
// slice 切片的底层结构本质上是一个对底层数组的引用和描述 
type slice struct {
	// 指向底层数组的地址
	array unsafe.Pointer
	// 切片长度 - 逻辑意义上slice中实际存放了多少元素
	len   int
	// 切片容量 - 物理意义上为slice分配了足够用于存放多少个元素的空间 
	// 使用slice时 要求cap永远大于等于len
	cap   int
}

// 访问的时候 索引是否越界 取决于len 例如 len = 3 访问s[3] 会越界
// 切片的类型定义如上，称之为slice header， 每个slice header 中存放的是内存空间的地址（array 字段），后续在传递切片时，
// 相当于对slice header进行了一次值拷贝，但内部存放的地址是相同的，因此对于slice本身属于引用传递操作。
```

##### slice的值传递
```go
s := make([]int, 5, 10)
s1 := [3:]
func changeSlice(s1 []int) {
	// 这里接收的s1 是一个新的slice header 但内容是一样的
	// 所以修改这里局部s1 会被外部方法中s1的值也造成影响
	// 但是如何修改局部中的s1 的len和cap 并不会对外部方法中的s1的len和cap造成影响
	s1[0] = 10
}
```

#### 初始化
```go
var s []int // 只是声明了slice的类型 并没有执行初始化操作 

// 这里切片的len和cap同时设置为8 切片的长度一旦被指定 
// 就代表对应位置已经分配了元素 即对应元素类型下的零值
s := make([]int,8) 

// cap >= len，在index为[len, cap) 的范围内，虽然内存空间已经分配，
// 但是逻辑意义上不存在元素，直接访问，会panic报数组访问越界
s := make([]int,8,16) 

// 初始化连带赋值 slice的len和cap都是3
s := []int{2, 3, 4}
```

##### 切片初始化源码
```go
// makeslice 函数用于创建一个新的切片，返回指向切片底层数组的指针
// 参数:
//   - et: 切片元素的类型描述符
//   - len: 切片的长度
//   - cap: 切片的容量
// 返回值:
//   - unsafe.Pointer: 指向切片底层数组的指针
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	// 根据 cap 结合每个元素的大小，计算出底层数组所需的总内存空间
	mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
	// 检查内存分配是否会导致溢出，或超出最大分配限制，或长度/容量参数无效
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// 二次检查长度参数，确保在容量检查失败时也能提供明确的错误信息
		mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen() // 触发长度参数错误的panic
		}
		panicmakeslicecap() // 触发容量参数错误的panic
	}

	// 调用垃圾回收器控制的内存分配函数，分配指定大小的内存空间
	// 参数:
	//   - mem: 要分配的内存大小
	//   - et: 内存中存储的数据类型
	//   - true: 表示分配的内存需要被清零
	return mallocgc(mem, et, true)
}
```

**引用传递** 将实例的地址信息传递到方法中，这样在方法中会直接通过地址追溯到实例所在位置，因此执行一些修改会直接影响到原实例
**值传递** 对实例进行一轮拷贝，得到一个副本，副本本身和实例是相互独立的

##### 内容截取
进行slice内容的截取，形如 s[a:b] 
a缺省则默认是0，比如s[:b]等价于 s[0:b]
b缺省，则默认填len(s) 比如s[a:]等价于 s[a:len(b)]
a和b都缺省，代表截取整个切片长度的范围，比如s[:]等价于 s[0:len(b)]

对切片slice进行截取操作，本质是一次引用操作，因为不论如何截取，底层复用的都是同一块内存空间中的数据，
只不过，截取操作会创建一个新的 slice header 实例。

---
##### 元素追加
**append操作，可以在slice末尾，额外新增一个元素，这里末尾指的是针对slice的长度len而言，**
**这个过程中倘若发现slice的剩余容量已经不足了，则会对slice进行扩容。**
```go
func Test_slice(t *testing.T){
	// 这里声明了长度和容量均为5的slice，这五个元素已经被填充为了零值
	s := make([]int, 5) 
	for i:= 0; i < 5; i++ {
		s = append(s, i)
	}
	// 结果为 s: [0,0,0,0,0,1,2,3,4]
}

// 正确示例一
func Test_slice(t *testing.T){
	s := make([]int,0,5)
	for i:=0; i < 5; i++ {
		s = append(s, i)
	}

	// 结果为 s: [0,1,2,3,4]
	// 这里也不会扩容
}

// 正确示例二
 func Test_slice(t *testing.T){
	s := make([]int,0,5)
	for i:=0; i < 5; i++ {
		s[i] = i
	}

	// 结果为 s: [0,1,2,3,4]
	// 这里也不会扩容
}
```


##### 切片扩容
当slice当前的长度len与容量cap相等时，下一次append会引发一次切片扩容
```go

// len:4 cap:4
s := []int{2, 3, 4, 5}
// len:5 cap:8
s = append(s, 6)
```

**扩容后的地址变化**
1. 容量足够时（cap-len > 0），内存地址不变
2. 需要扩容时，内存地址会改变，切片的Data指针指向新数组

扩容流程源码 routime/slice.go 文件中的growslice方法中
```go
// growslice 扩容方法
// oldPtr: 指向原切片底层数组的指针
// newLen: 扩容后需要的长度（原长度 + 新增元素数量）
// oldCap: 原切片的容量
// num: 新增元素的数量
// et: 切片元素类型的元信息
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	oldLen := newLen - num // 原切片长度

	// ...

	if newLen < 0 {
		panic(errorString("growslice: len out of range"))
	}

	if et.Size_ == 0 {
		
		return slice{unsafe.Pointer(&zerobase), newLen, newLen}
	}

	// 计算新容量
	newcap := nextslicecap(newLen, oldCap)

	// ...
	
	// 内存分配
	var p unsafe.Pointer
	if !et.Pointers() {
		p = mallocgc(capmem, nil, false)
		
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(oldPtr), lenmem-et.Size_+et.PtrBytes, et)
		}
	}
	// ...

	// 数据迁移
	memmove(p, oldPtr, lenmem)

	return slice{p, newLen, newcap}
}

// nextslicecap 扩容后新容量的核心函数
// newLen：扩容后需要的长度（当前长度+新增元素数量）
// oldCap：当前切片的容量
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	// 1. 扩容后的总长度超过两倍容量，直接返回该长度
	// 比如当前有20个元素，容量是50，需要200个元素，20+200 > 2*50，那么直接返回220
	if newLen > doublecap {
		return newLen
	}
	// 2. 小切片处理
	const threshold = 256
	// 当前切片容量 < 256，返回两倍当前容量
	// 比如当前容量是50，那么直接返回100
	if oldCap < threshold {
		return doublecap
	}
	// 3. 大切片处理
	// 
	for {
		// (当前容量+3*256) / 4 
		newcap += (newcap + 3*threshold) >> 2

		// 当容量 >= 扩容后的长度 则返回
		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	// 4. 溢出检查 
	// 容量非正，则返回所需的长度
	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

**扩容机制总结**：
1. 扩容后的总长度超过两倍容量，直接返回所需容量
2. 如果当前容量小于 256，则新容量直接翻倍
3. 如果当前容量大于等于 256，则新容量会按照一个渐进的增长率（约 1.25x）增加

**slice不是并发安全的数据结构，在使用时需要注意并发安全问题**

#### 元素删除
从切片中删除元素的实现思路，本质上和切片内容截取的思路是一致的。
```go
// 删除切片中的中间元素
func Test_slice(t *testing.T){
	s := []int{0,1,2,3,4}
	// 删除index=2的元素
	s = append(s[:2],s[3:])
	// s = [1,2,3,4]
}

// 删除切片中的所有元素
func Test_slice(t *testing.T){
	s := []int{0,1,2,3,4}
	s = s[:0]
	// s: [], len:0, cap:5
}
//这样操作后slice header中的指针array仍指向原处，
// 但是逻辑意义上其长度len已经等于0，而容量cap仍保留为原值

// 切片拷贝
func Test_slice(t *testing.T){
	s := []int{1, 2, 3, 4}
	s1 := s
	// 简单拷贝 s和s1的地址是一致的
}
// 完成复制
func Test_slice(t *testing.T){
	s := []int{1, 2, 3, 4}
	s1 := make([]int, len(s))
	copy(s1, s)
}
```

在实现上，slice的完整复制可以调用系统方法copy，s和s1的地址是相互独立的