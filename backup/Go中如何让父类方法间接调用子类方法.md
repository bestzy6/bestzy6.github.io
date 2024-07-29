# 前言

在一些问题场景中，我们希望使子类继承父类的方法，来减少了代码冗余，使得拓展的子类仅需实现父类的某些方法即可实现功能。这种做法在“模板方法”设计模式中经常见到。模板方法在超类中定义了一个算法的框架， 允许子类在不修改结构的情况下重写算法的特定步骤。

Go并不像Java和Python，不存在抽象类的概念，因此无法直接通过子类实现父类抽象类的方式来实现“模板方法”。

# 案例

在以下案例中，我们定义了一个Father类、一个Son类和一个Streamer接口。

```go
type Streamer interface {
    Call() string
    CallStream() []string
}

// ---

type Father struct {
}

func (f *Father) Call() string {
    stream := f.CallStream()
    return strings.Join(stream, "")
}

func (f *Father) CallStream() []string {
    panic("need to implement it")
}

// ---

type Son struct {
    Father
}

func (s *Son) CallStream() []string {
    return []string{
        "a",
        "b",
        "c",
    }
}
```

在以下代码中，我们期望son.Call()执行father.Call()方法的逻辑，返回“abc”。

```go
func main() {
    son := Son{}
    son.Call()
}
```

然而，实际执行的结果却是Panic。

```
panic: need to implement it

goroutine 1 [running]:
main.(*Father).CallStream(...)
```

因为Father.Call()方法中调用的函数是Father.CallStream()，而并非Son的。

在 Go 语言中，由于语言特性的限制，无法像 Java 中那样直接调用父类的公共方法并间接调用子类实现的抽象方法。这与Java、Python等面向对象语言不同，例如在Python中，son.Call()是能够执行的，而且能够返回我们期望的“abc”。代码如下：

```python
class Father:
    def stream(self):
        arr = self.streamCall()
        return "".join(arr)

    def streamCall(self):
        pass


class Son(Father):
    def streamCall(self):
        return ['a','b','c']


if __name__ == "__main__":
    son = Son()
    ans = son.stream()
    print(ans) // abc
```

# 方法-接口与组合

让Father组合Streamer接口，然后将Call中原本调用的CallStream修改为调用Streamer的CallStream方法。

```go
type Father struct {
    Streamer
}

func (f *Father) Call() string {
    stream := f.Streamer.CallStream()  // 这里修改为调用Streamer的CallStream方法
    return strings.Join(stream, "")
}

func (f *Father) CallStream() []string {
    panic("need to implement it")
}

type Son struct {
    Father
}

func (s *Son) CallStream() []string {
    return []string{
        "a",
        "b",
        "c",
    }
}

func main() {
    son := Son{}
    son.Father.Streamer = &son // 为Father.Streamer赋值为son
    res := son.Call()
    fmt.Println(res) // abc
}
```

# 总结

本文主要介绍了如何在Go语言中实现“模板方法”设计模式。首先介绍了在Go语言中无法直接通过子类实现父类抽象类的方式来实现“模板方法”的限制，随后通过接口与组合的方式实现了类似于Java、Python等面向对象语言中的“模板方法”。最终给出了Go语言中实现“模板方法”的具体例子。

