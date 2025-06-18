## Golang单机锁
**单机锁**：指在单个机器或单个进程内用于控制并发访问的同步机制。


``` go
// 并发读
// 读 + 写、 写 + 写

// 使用Mutex来控制map的并发访问
type MyConcurrentMap struct {
	sync.Mutex
	mp map[int]int

}
// NewMyConcurrentMap 构造函数
func NewMyConcurrentMap() *MyConcurrentMap {
	return &MyConcurrentMap {
		mp: make(map[int]int),
	}
}
func (m *MyConcurrentMap)Get(key int) (int, bool) {
	// Lock之前 并发、自由
	m.RWMutex.RLock()
	// 一次只有一个 goroutine 访问
	v, ok := m.mp[key]

	m.RWMutex.RUnlock()

	// Unlock之后 并发、自由
	return v, ok
}
func (m *MyConcurrentMap)Put(key, val int) {
	m.Mutex.Lock()

	m.mp[key] = val

	m.Mutex.Unlock()
}
func (m *MyConcurrentMap)Delete(key int) {
	m.Mutex.Lock()

	delete(m.mp, key)

	m.Mutex.Unlock()

}
```
---

#### 互斥锁（Mutex）
**作用**：保护共享资源，防止多 goroutine 并发访问导致数据竞争（Race Condition）。  
**特点**：
- 独占访问：同一时间仅允许一个 goroutine 持有锁，读写操作均需获取独占锁。
- 适用场景：读写操作频率相近或均较少的场景。
#### 读写分离锁（RWMutex）
- **读锁（RLock）**：共享锁，允许多个 goroutine 同时持有（适用于并发读）。
- **写锁（Lock）**：独占锁，同一时间仅允许一个 goroutine 持有（适用于写操作）。  
**特点**：
- 读操作不阻塞读操作，但写操作会阻塞所有读写操作。
- 适用场景：多读少写的高并发场景。

**Mutex数据结构**
``` go
type Mutex struct {
	state int32 // 锁状态字段，通过不同bit位存储标识状态
	sema  uint32 // 信号量，用于阻塞和唤醒goroutine
	
}
```

**状态位常量定义**
``` go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	starvationThresholdNs = 1e6
)
```


- **左移 <<** ： 数对应的二进制位全部向左移动若干位，高位丢弃，低位补 0，公式：x << n = x * 2^n 
- **右移 >>** ：快速除以2的幂次方  x >> n = x / (2^n)
- **按位与 (&)**：按位与运算将参与运算的两数对应的二进制位相与，当对应的二进制位均为 1 时，结果位为 1，否则结果位为 0
- **按位或 (|)** ：按位或运算将参与运算的两数对应的二进制位相或，只要对应的二进制位中有 1，结果位为 1，否则结果位为 0
- **按位异或(^)**：按位异或运算将参与运算的两数对应的二进制位相异或，当对应的二进制位值不同时，结果位为 1，否则结果位为 0。

**state字段位布局**
- mutexLocked = 1：state最右侧的一个bit位标志是否上锁，0-未上锁，1-已上锁
- mutexWoken = 2：state右数第二个bit位标志是否有goroutine从阻塞中被唤醒，0-没有，1-有
- mutexStarving = 4：state右数第三个bit位标志Mutex是否处于饥饿模式，0-正常，1-饥饿
- mutexWaiterShift = 3：表示右侧存在3个bit位标识特殊信息，分别为上述的mutexLocked、mutexWoken、mutexStarving
- starvationThresholdNs = 1ms：sync.Mutex进入饥饿模式的等待时间阙值

高29位的值聚合位一个范围为0~2^29-1的整数，表示在阻塞队列中等待的协程个数

---

**Lock方法源码**
``` go
// Lock快速出口
func (m *Mutex) Lock() {
	// 尝试通过原子操作直接将 state 从 0 改为 1（快速路径）
	// 原子操作是通过硬件或操作系统提供的原子指令，将检查-修改合并为一个不可分隔的操作
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	m.lockSlow()
}
// Mutex.lockSlow()方法
// 几个局部变量
func (m *Mutex) lockSlow() {
	var waitStartTime int64 // 记录当前goroutine开始等待锁的时间戳，单位ns
	starving := false // 当前锁是否处于饥饿模式
	awoke := false // 当前goroutine是否被唤醒
	iter := 0 // 当前goroutine自旋次数
	old := m.state // 保存当前Mutex状态的旧值
	
	// ...
}

```

**lockSlow方法源码**
```go
// lockSlow 慢出口
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	// 1. 自旋空转
	for {
		// 判断锁是否已锁定并未处于饥饿模式 但仍满足自旋 
		// 必须处于正常模式 饥饿模式下不能自旋，必须排队
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter){
			
			// !awoke - 当前协程未被其他协程显式唤醒
			// old&mutexWoken == 0 锁未标记其他唤醒者（避免重复设置）
			// old>>mutexWaiterShift != 0 存在等待者（否则无需设置唤醒标志）
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				// old|mutexWoken 将mutexWoken的值改为1
				// 当前m.state的值是否等于old（锁的状态未被其他协程修改）
				// 相等 将 m.state 更新为 old | mutexWoken
				//（即设置 mutexWoken 标志位）
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				// 将当前协程标记为 唤醒状态
				awoke = true
			}
			runtime_doSpin() // 自旋
			iter++
			old = m.state // 重新加载锁的状态
			continue
		}
		// 2. 预写状态值 - 自旋条件不满足时（进入饥饿模式、自旋次数超限）走进该流程
		new := old
		// old&mutexStarving == 0 ：处于正常模式
		if old&mutexStarving == 0 {
			// 完整写法：new = new | mutexLocked
			// 通过位运算将 mutexLocked 标志位（值为 1）设置到变量 new 中
			// 抢锁成功
			new |= mutexLocked
		}
		// old锁已被占有或是饥饿状态
		if old&(mutexLocked|mutexStarving) != 0 {
			// new = new + 1 << mutexWaiterShift
			// 1 << mutexWaiterShift 也就是 0001000
			// 增加一个等待锁的goroutine
			new += 1 << mutexWaiterShift
		}
		// 当前 goroutine 是否已等待超过 starvationThresholdNs（1ms）
		// 并且锁仍被其他goroutine持有
		if starving && old&mutexLocked != 0 {
			// 修改锁的模式为饥饿模式
			new |= mutexStarving
		}
		// 当前goroutine被唤醒
		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			// new = new &^ mutexWoken
			// 清除new中的mutexWoken标志位
			new &^= mutexWoken
		}
		// 3. 加锁&阻塞&苏醒
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			// 判断锁是否真正加锁成功	
			if old&(mutexLocked|mutexStarving) == 0 {
				break // 锁未被锁定且不处于饥饿模式，当前协程成功获取锁
			}

			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				// 设置当前时间
				waitStartTime = runtime_nanotime()
			}
			// 挂起前处理 ...
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			// 从阻塞队列被唤醒了
			// 通过当前时间-waitStartTime与starvationThresholdNs的比较 
			// 判断是否处于饥饿模式
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			// 锁处于饥饿模式
			if old&mutexStarving != 0 {
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				// 将锁标记为已锁定 并减少一个等待者计数
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 若当前协程是最后一个等待者 或已不再饥饿 则退出饥饿模式
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break // 成功获取锁 跳出循环
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
			
		}
	}
}
```

**Unlock方法源码**
```go
// Unlock - 释放一个互斥锁
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}
	// 原子性地将锁状态减去mutexLocked（值为1），即清除"锁已占用"标志
	new := atomic.AddInt32(&m.state, -mutexLocked)
	// new不等于0 说明有其他等待的goroutine或其他情况
	if new != 0 {
		m.unlockSlow(new)
	}
}

```

**unlockSlow方法源码**
```go
// unlockSlow
func (m *Mutex) unlockSlow(new int32) {
	// 1. 未加锁的异常情形
	// 对未加锁的互斥锁进行解锁 直接抛出fatal
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}
	// 2. 非饥饿模式的处理
	if new&mutexStarving == 0 {
		old := new
		for {
			// 阻塞队列中等待者为空、锁已被持有、有goroutine被唤醒、处于饥饿模式
			// 这些情况不需要唤醒其他goroutine 
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// 准备唤醒一个等待者 
			// 1<<mutexWaiterShift 减少一个等待者 
			// 设置mutexWoken标志 表示唤醒一个goroutine
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			// 使用CAS原子操作更新状态
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				// 唤醒一个goroutine
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			// 如果CAS操作失败 将新状态更新到old 并重试
			old = m.state
		}
	} else {
		// 3. 饥饿模式下 直接唤醒队列头部的goroutine即可
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

---

### Sync.RWMutex 读写锁，可以理解为一把读锁加一把写锁
- 写锁具有严格的排他性，当其被占用，其他试图取写锁或读锁的goroutine均阻塞
- 读锁具有有限的共享性，当其被占用，试图取写锁的goroutine会阻塞，试图取读锁的goroutine可与当前goroutine共享读锁

**RWMutex数据结构**
```go
type RWMutex struct {
	w           Mutex        // 互斥锁
	writerSem   uint32       // 阻塞等待写锁的goroutine队列
	readerSem   uint32       // 阻塞等待读锁的goroutine队列

	// 正值：记录当前活跃的读者数量（正在执行读操作的 goroutine 数）
	// 负值：readerCount 值的含义
	// -rwmutexMaxReaders + N	有写操作等待，且还有 N 个活跃读者未退出（N >= 0）。
	// -rwmutexMaxReaders	写操作等待，且所有活跃读者已退出（N = 0）。
	readerCount atomic.Int32 
	// 记录写操作等待的读者数量（在写操作请求锁时尚未完成的读者数）
	readerWait  atomic.Int32 
}
// 共享读锁的goroutine数量上限 值为 2^29
const rwmutexMaxReaders = 1 << 30
```

#### 读锁流程

**RLock方法源码**
```go
// RLock 读锁操作
func (rw *RWMutex) RLock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// 增加读者计数 检查是否需要阻塞（需要阻塞 说明目前有写锁）
	if rw.readerCount.Add(1) < 0 {
		runtime_SemacquireRWMutexR(&rw.readerSem, false, 0) // 阻塞读操作
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}
```

**RUnlock方法源码**
```go
// RUnlock 解读锁
func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		_ = rw.w.state
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
	// 减少读者数量 检查是否需要唤醒写者
	if r := rw.readerCount.Add(-1); r < 0 {
		rw.rUnlockSlow(r)
	}
	if race.Enabled {
		race.Enable()
	}
}
```

**rUnlockSlow方法源码**
```go
// rUnlockSlow 方法
func (rw *RWMutex) rUnlockSlow(r int32) {
	// r+1 == 0：防止对未加锁的 RWMutex 调用 RUnlock()
	// r+1 == -rwmutexMaxReaders 表示之前没有加过读锁 只有写锁
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		fatal("sync: RUnlock of unlocked RWMutex")
	}
	// 检查是否为最后一个退出的读者，若是则唤醒写者
	if rw.readerWait.Add(-1) == 0 {
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

---

#### 写锁流程

**Lock方法源码**
```go
// Lock方法 加写锁
func (rw *RWMutex) Lock() {

	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	rw.w.Lock() // 获取互斥锁 确保写操作的排他性

	// 原子地将 readerCount 减去 rwmutexMaxReaders（1 << 30），
	// 使其变为负数 - 这个操作会修改rw.readerCount本身的值
	
	// 计算当前活跃读者数 r - 这里不会修改rw.readerCount的值，
	// 而是+rwmutexMaxReaders 然后赋值给r
	// 将readerCount的值设为负值 阻止新的读操作进入
	r := rw.readerCount.Add(-rwmutexMaxReaders) + rwmutexMaxReaders
	// r != 0：存在读者 
	// rw.readerWait.Add(r) != 0：设置需要等待的读者数量（readerWait = r）
	// 当有读操作时 写操作被阻塞到这个信号量上 等待所有读操作完成
	if r != 0 && rw.readerWait.Add(r) != 0 {
		// 阻塞到writerSem
		runtime_SemacquireRWMutex(&rw.writerSem, false, 0)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
```

**Unlock方法 解写锁**
```go
func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}
	// readerCount恢复为正值
	r := rw.readerCount.Add(rwmutexMaxReaders)
	// 没有加过写锁 或者 读者数量达到了上限 也就是rwmutexMaxReaders
	if r >= rwmutexMaxReaders {
		race.Enable()
		fatal("sync: Unlock of unlocked RWMutex")
	}
	// 唤醒所有的读锁
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}

	rw.w.Unlock() // 释放互斥锁

	if race.Enabled {
		race.Enable()
	}
}
```
