**Context**用于跨 goroutine 传递请求作用域的数据、取消信号、超时和截止时间。它主要用于**控制并发任务的执行**
#### Context数据结构
``` go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```
Context 为 interface，定义了四个核心的api：
- Deadline：返回context的过期时间；
- Done：返回context中的channel；
- Err：返回错误；
- Value：返回context中的对应key的值。
#### Context的实现类
##### emptyCtx
``` go
// emptyCtx 永远不会被取消，没有值，也没有截止日期。
// 它是 backgroundCtx 和 todoCtx 的公共基类。
type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {  
    return  
}  
  
func (emptyCtx) Done() <-chan struct{} {  
    return nil  
}  
  
func (emptyCtx) Err() error {  
    return nil  
}  
  
func (emptyCtx) Value(key any) any {  
    return nil  
}

```
emptyCtx是一个空的context，本质上是一个struct{}；
- Deadline方法返回一个公元元年以及false的flag，标识当前context不存在过期时间；
- Done方法返回一个nil值，用户无论往nil中写入或读取数据，均会陷入阻塞；
- Err方法返回的错误永远为nil；
- Value方法返回的value也永远为nil。
##### context.Background() & context.TODO()
``` go
type backgroundCtx struct{ emptyCtx }

type todoCtx struct{ emptyCtx }

func Background() Context {  
    return backgroundCtx{}  
}
func TODO() Context {  
    return todoCtx{}  
}
```
Background()和TODO()方法返回的均是emptyCtx类型的一个实例。

##### cancelCtx
``` go
// cancelCtx 可取消的Context类型
type cancelCtx struct {  
    Context // 继承父context的方法
  
    mu       sync.Mutex            
    done     atomic.Value // 存储一个 <-chan struct{}      
    children map[canceler]struct{} // 记录所有子canceler
    err      error
    cause    error
}
// canceler 
type canceler interface {  
    cancel(removeFromParent bool, err, cause error) // 取消操作
    Done() <-chan struct{} // 返回监听取消的channel
}
```

##### context.WithCancel()

