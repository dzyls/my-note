### 协程

---

协程就是用户级线程。

Java语言创建线程要涉及到用户态系统态切换，而Go语言创建协程是不会涉及到用户态和系统态的切换的。

Go语言创建协程很简单 ：`go`关键字。示例 :

```go
var strChan = make(chan string)

var wg = sync.WaitGroup{}

func main() {
   wg.Add(1)
   go sendA()
   go sendB()
   wg.Wait()
}

func sendA() {
   for {
      s := <-strChan
      log.Println("A :" +s)
      strChan <- "A"
   }
}

func sendB() {
   for  {
      strChan <- "B"
      s := <- strChan
      log.Println("B :" + s)
   }
}
```

使用go关键字就可以创建线程。相比于Java语言创建线程的Thread/Runnable/Callable/FutureTask等，Go是十分的简单。

这里也体现了go语言的设计哲学，简单即是美，less is more。

回顾一下并发与并行 ：

> 并发 ：同一个时间段内，两个程序同时运行
>
> 并行 ：同一个时间点，两个程序同时运行
>
> 并行是并发的特殊情况。

协程相比与系统级线程开销很小，内存充足的情况下，上万协程不成问题。



### 协程的状态

---

值得一提的是协程的状态。

回顾Java线程的状态 ：

- NEW
- RUNNABLE : 包括Running和Ready状态，Running状态很短暂，没必要设置两个
- WAITING
- TIME_WAITING
- BLOCKED
- TERMINATE



而Go协程的状态有 ：

- 创建
- 运行
- 阻塞
- 退出

需要特别注意的是 ：

- `time.Sleep(time.Second * 2)`是**运行状态**而不是阻塞状态
- Go的阻塞状态不会自己终止，需要其他协程来通知它要退出阻塞状态
- 如果没有其他协程通知要退出阻塞状态，那么这个协程将会永远等待下去。程序会被视为死锁，程序死锁会引起程序崩溃。



### 协程调度

---

Go的协程调度MPG模型。

M表示系统线程

P表示逻辑处理器、

G代表协程



大多数调度是由逻辑处理器来完成的，逻辑处理器将不同的协程交给系统线程进行处理。

一个协程执行片刻后，会自发的让出系统线程，让别的协程也获得机会执行。



### defer延迟调用

---

defer是延迟调用的关键字。

越在前面的defer，执行顺序越靠后。

事实上，defer有两个执行堆栈 ：

- 一个是普通的堆栈，按顺序执行
- 一个是defer堆栈，倒序执行

```java
func main() {

   defer fmt.Println("3")
   defer fmt.Println("2")
   defer fmt.Println("1")
   defer fmt.Println("0")

}
// 将输出0123
```



### defer可以修改返回值

---

```java
func main() {

   fmt.Println(add(5))

}

func add(num int) (r int) {
   defer func() {
      // 此处覆盖了r的值，因此返回15
      r += num
   }()
   // 此处等同于 r = 5 + num
   return 5 + num
}
```

如上，defer会修改了返回值



### 恐慌和恢复

---

Go不支持异常，但有自己的一套恐慌和恢复机制。

`panic` 可以制造一恐慌

`recover`可以恢复

如果一个恐慌没有消除，那么将蔓延到其他协程，造成整个程序的崩溃。

paninc可以使用defer来确保即使异常也关闭资源

```java
func main() {

   defer func() {
      // recover可以接收panic
      v := recover()
      if v != nil {
         fmt.Println("panic :",v)
      }
      // 可以兜底来关闭资源
   }()
   panicTest()

}

func panicTest() {
   fmt.Println("Hello")
   panic("byebye!")
   fmt.Println("Can't Reachable")
}
```

