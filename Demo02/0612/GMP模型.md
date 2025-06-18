
##### 协程：又称为用户级线程
1. 与线程存在映射关系，为M：1；
2. 创建、销毁、调度在用户态完成，对内核透明，所以更轻；
3. 从属同一个内核级线程，无法并行；一个协程阻塞会导致从属同一个线程的所有协程无法执行。
##### Goroutine：经Golang优化后的特殊“协程”
1. 与线程存在映射关系，为M：N；
2. 创建、销毁、调度在用户态完成，对内核透明，所以更轻；
3. 可利用多个线程，实现并行；
4. 通过调度器，可以实现线程间的动态绑定和灵活调度；
5. 栈空间大小可以动态扩展，因地制宜。
---
#### GMP模型
GMP = goroutine + machine + processor
###### 1. G
- G即goroutine，是golang中对协程的抽象；
- G有自己的运行栈、状态、以及执行的任务函数（用户通过 go func（） 指定）；
- G需要绑定到P才能执行，在G的视角中，P就是它的cpu。
###### 2. M
- M即machine，是golang中对线程的抽象；
- M不能直接执行G，需要先和P绑定，由其实现代理。
###### 3. P
- P即processor，是golang中的调度器；
- P的数量决定了G最大并行数量，用户可以通过GOMAXPROCS进行设定（超过CPU核后则没有意义）。
---
#### 核心数据结构
``` Go

type g struct {
	// ...
	m        *m // 在p的代理下，负责执行当前g的m
	// ...
	sched    gobuf
	// ...
}

// gobuf 保存goroutine的执行状态
type gobuf struct {  
	sp      uintptr
    pc      uintptr  
    g       guintptr  
    ctxt    unsafe.Pointer  
    ret     uintptr  
    lr      uintptr  
    bp      uintptr 
}

type m struct {
	g0      *g // g0：负责执行g之间的切换调度
	// ...
	tls     [tlsSlots]uintptr
}

type p struct {
	// ...
	runqhead uint32 // 队列头部
	runqtail uint32 // 队列尾部
	runq     [256]guintptr // 本地goroutine队列，最大长度为256

	runnext guintptr // 下一个可执行的goroutine
	// ...
}

// schedt 全局队列
type schedt struct {
	// ...
	lock     mutex // 操作全局队列时的锁
	// ...
	runq     gQueue // 全局goroutine队列
	runqsize int32 // 队列的容量
	// ...
}
```

#### 两种g的转换
1. g0通过func gogo()来调度g
2. g通过func m_call()来切换为g0

---
#### 调度流程的源码
调度流程的主干方法为schedule函数，此时的执行权位于g0手中
``` go
func schedule() {
	// ...
	// 寻找下一个执行的goroutine
	gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available
	// ...
	// 执行该goroutine
	execute(gp, inheritTime)
}


```


