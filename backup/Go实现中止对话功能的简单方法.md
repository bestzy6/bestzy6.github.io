# 背景

开发LLM应用往往需要使用流式输出，而在LLM对话的场景下，一个常见的功能是用户手动停止输出，本文将介绍一种简单的手动停止输出的实现方法。

# 实现

实现流式接口通常使用SSE协议与WebSocket协议，无论使用哪种协议，“流式输出”与“监听终止”都属于两个不同的协程。

![image](https://github.com/user-attachments/assets/f66f9098-3008-40f6-9b22-a145e57fcef9)

在 Go 语言中，`context.Context` 对象可以实现取消、超时、截止日期等功能。因此基于`context.Context`库，我们可以设计一个状态保存器，称之为cancelMap。

cancelMap中记录正在进行输出的协程的cancelFunc。



```go
type Done func()
type WaitDone func()

type cancel struct {
    cancelFun  context.CancelFunc
    doneNotify chan struct{}
}

type CancelMap struct {
    m    map[string]cancel
    lock sync.RWMutex
}

func NewCancelMap() *CancelMap {
    return &CancelMap{
        m: make(map[string]cancel),
    }
}

func (c *CancelMap) SetCancel(key string, cancelFunc context.CancelFunc) Done {
    c.lock.Lock()
    defer c.lock.Unlock()

    // 创建一个通道，以便将来可以通知未完成的 goroutine
    done := make(chan struct{}, 1)
    c.m[key] = cancel{
        cancelFun:  cancelFunc,
        doneNotify: done,
    }

    // 返回 Done 函数，以便稍后取消该 goroutine
    return func() {
        c.lock.Lock()
        defer c.lock.Unlock()
        // 通知 goroutine 执行终止动作，关闭通道
        close(done)
        // 从 map 中删除取消函数和通知信号
        delete(c.m, key)
    }
}

func (c *CancelMap) GetCancel(key string) (context.CancelFunc, WaitDone) {
    c.lock.RLock()
    defer c.lock.RUnlock()

    canc, ok := c.m[key]
    if !ok {
        // 如果不存在cancelFunc，则返回空
        return nil, nil
    }

    return canc.cancelFun, func() {
        <-canc.doneNotify
    }
}
```

以上代码定义了一个 CancelMap 类型，其中包含一个 map，用于存储取消函数和通知完成的 channel。同时也定义了 Done 和 WaitDone 类型用于返回取消函数和等待完成的函数的函数类型。

CancelMap 提供了 SetCancel 和 GetCancel 方法。SetCancel 方法用于将取消函数和已完成的通知存储到 map 中，并返回一个 Done 函数，以便稍后使用这个函数取消并从 map 中删除该条目。GetCancel 方法用于从 map 获取存储的取消函数和已完成的通知，并返回一个 context.CancelFunc 类型的取消函数和一个 WaitDone 函数，用于等待已完成的通知。

在内部，CancelMap 使用互斥锁来保证对 map 的并发访问的安全性。同时，每个 cancel 实例存储了一个 doneNotify channel，用于向等待该条目完成的 WaitDone 函数发送信号。返回的 WaitDone 函数会从 doneNotify 中接收信号，以便在该函数完成后恢复执行。

# 示例

> 此部分为伪代码

流式输出协程

```go
//...    
    ctx, cancelFunc := context.WithCancel(context.Background())
    cancelMap := NewCancelMap()
    done := cancelMap.SetCancel("k", cancelFunc)

    go func() {
        // 结束之前执行done()
        defer done()

        for {
            select {
            case <-ctx.Done():
                fmt.Println("stop by user")
                return
            default:
                break
            }

            // 获取stream片段
            response, _ := stream.Recv()
            fmt.Println(response.Choices[0].Delta.Content)
        }
    }()
```

监听终止命令协程

```go
// ...
    go func() {
        // ...
        
        // 获取cancelFunc
        f, waitDone := cancelMap.GetCancel("k")
        if f != nil {
            // 执行cancelFunc
            f()
        }

        // 等待流式输出协程执行完毕
        if waitDone != nil {
            waitDone()
        }

        // ... 
    }()
```

# 总结

本文提供了一种使用go语言实现终止流式输出的简单方法。总的来说，终止流式输出的逻辑本质上是多个协程的同步问题，而go原生适合开发并发功能，使用go的channel、context等机制能够达到目的。