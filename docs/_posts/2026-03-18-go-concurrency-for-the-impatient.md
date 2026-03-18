---
layout: post
title: Go Concurrency for the Impatient
date: 2026-03-18 15:07 +0800
---

**并发**（concurrency）是指 CPU 同时处理多个任务的能力。单核 CPU 的物理限制决定了其一次只能只能执行一条指令，也就只能处理单个任务。并发实际上是通过极速的任务切换实现的。并发机制同步任务处理，最大化硬件利用率以构建**高性能**以及**响应式**程序。并发是一切现代软件之基石。

![image.png](/assets/images/260318-go-concurrency/image.png)

**并行**（parallel）需要多核 CPU 为硬件基础，真正地在同一时间独立地执行多个任务：

![image.png](/assets/images/260318-go-concurrency/image%201.png)

并发为**逻辑**上的同时，而并行是**物理**上的同时。

并发编程难且在其过程中容易犯错。要写出好的并发程序，除了需要理解并发的本质外， 还需要丰富的经验。现代编程语言通常提供强大且对用户友好的**并发原语（concurrency primitive）**—— Go 提供了 **goroutine** 以及 **channel**。

# 1 进程、线程以及协程

## 1.1 进程

**进程**（process）是 OS 提供给用户的最基本的抽象机制。一个进程就是一个正在运行的程序。程序本身只是存储在硬盘中的一些指令以及静态数据的集合，当 OS 加载并执行这个集合时，就创建了一个进程。

OS 通过 CPU 的**时分**（time sharing）机制以及**上下文切换**（context switch）支持多个进程的并发。OS 决定何时进行上下文切换，也决定下一个切换的进程是哪个，这个过程称作**调度**（scheduling）。高效和安全的并发需要合理的调度策略。

一个程序运行时，除了其本身的指令和静态数据，还依赖于一些**机器状态**：分配给它的内存空间、CPU 的寄存器、程序计数器的位置等等。这一切的总和构成了一个进程。

创建进程需要分配独立的**地址空间**（address space），这些内存空间专属于一个线程，其他线程无法触及：

![image.png](/assets/images/260318-go-concurrency/image%202.png)

**进程控制块**（process control block，PCB）是 OS 用来表达一个进程的数据结构，其中包含了进程的全部信息，同时也是上下文切换时，保存和读取进程的硬件执行状态（PC、SP 等）的地方。PCB 中包括了**进程 ID**（PID）、执行状态、硬件状态、调度信息等。OS 切换进程时，首先保存当前进程的状态（将 CPU 中的执行状态存入对应 PCB），然后从对应 PCB 中加载下一个进程的状态。

## 1.2 线程

进程的问题在于创建时分配的地址空间**开销大**，而且切换时保存和读取的状态也多。并发度高时，性能存在明显上限。独立的内存空间也使得**进程间通信**的复杂度提高。

**线程**（thread）是 OS 能够进行**运算调度**的最小单位。线程包含在进程中，是进程中的实际运作单位。一个进程可包含多个线程，它们共享这个进程的**代码段**、**数据段**（全局变量、静态数据）、**堆**（动态分配的内存）。每个线程拥有其独立的**寄存器状态**（包括 PC）、**栈**（局部变量和函数调用信息）、**线程 ID**。

线程创建和上下文切换的开销都要小得多，线程间进行通信也更方便。

## 1.3 协程

线程调度是操作系统内核的工作。硬件时钟每隔数毫秒中断处理器，唤醒内核调度器，挂起当前线程，将其寄存器存入内存，决定下一个执行的线程，将下一个线程的寄存器从内存中恢复到处理器中，唤醒处理器。上述将处理器控制权移交给另一个线程的全过程就是一次上下文切换。简而言之：保存当前线程状态；恢复下一个线程状态；更新调度器数据结构。上述过程在高并发环境下的大开销会成为性能瓶颈。

考虑一个电商系统，一条付款信息就会创建一个线程查询数据库。当并发连接数成千上万时，会有下列灾难性后果：

- 内存爆炸：每个线程的默认栈大小通常为 8MB，10000 个就会占用将近 20GB 内存。
- 切换开销巨大：上万个线程会导致 CPU 大量时间花在切换上。

---

Goroutine 是轻量化的、用户态的「微型线程」。Go 运行时维护一个调度器，使用 **m:n 调度**（m:n scheduling）—— m 个 goroutine 被多路复用到 n 个操作系统线程。Go 调度器和操作系统调度器的工作逻辑一致，只是 Go 调度器只关心单个 Go 程序的 goroutine 的调度。Go 调度器不会定时启动，而是由某些语言构造唤醒：如 `time.Sleep` 和 channel I/O、互斥锁导致的 goroutine 阻塞都会唤醒调度器。Goroutine 之间的上下文切换不必切换到操作系统内核态，因此 goroutine 直接的切换开销比操作系统线程的切换开销要小得多。操作系统线程的栈是固定大小的。Goroutine 的栈则是可成长的——初始大小一般为 2KB，上限大概为 1GB。

此外，goroutine 可以通过 `runtime.Gosched()` 主动让出；且运行超时的 goroutine 会被标记为抢占，并被强制切换。

Go 调度器使用参数 `GOMAXPROCS` 限制 Go 程序可同步使用的最大操作系统线程数，也是 m:n 调度中的 n 数。

`GOMAXPROCS` 的默认值是机器的 CPU 数。可以使用下列语句查询：

```go
fmt.Println(runtime.GOMAXPROCS(0)) // 0: query, not modify
```

可以通过附加 `GOMAXPROCS` 环境变量，或使用 `runtime.GOMAXPROCS` 函数来控制 `GOMAXPROCS` 的值：

```bash
$ GOMAXPROCS=1 go run main.go
```

> 并发编程将程序表达为数个**自主活动**的组合。
Concurrent programming is the expression of a program as a composition of several autonomous activities.
> 

**并发**机制同步任务处理，最大化硬件利用率以构建**高性能**以及**响应式**程序。并发是一切现代软件之基石。

并发编程困难且在其过程中容易犯错。要写出好的并发程序，除了需要理解并发的本质外， 还需要丰富的经验。现代编程语言通常提供强大且对用户友好的**并发原语（concurrency primitive）**—— Go 提供了 **goroutine** 以及 **channel**。

# 1 Goroutine

Goroutine 是 Go 的[**协程**](https://zh.wikipedia.org/zh-cn/%E5%8D%8F%E7%A8%8B)（coroutine）实现。Goroutine 轻量、无关于进程，且由 Go **运行时**系统管理。

Go 程序总以 `main` goroutine 开始，`main` goroutine 调用 `main` 函数。`go` 语句用以创建一个新的goroutine。



`go` **语句**

`go` 语句指任何以关键字 `go` 开头的函数或方法调用，表示在一个新的 goroutine 中执行这个函数/方法。与直接函数调用的区别在于——函数调用时，主 goroutine 必须等待函数返回；而通过 `go` 语句调用函数不会使主程序暂停。

```go
f()     // call f(), WAIT for return
go f()  // call f() in a new goroutine, DO NOT WAIT
```



**示例：Spinner**

Some activity like downloading files takes a long time. User may want to see some signals that the activity is still running, or they may want to see the progress as, for example, a progress bar. Showing the status, and the activity itself together, are **concurrent activities**. 

- The `spinner` is a simple example
    
    ```go
    // fib(n) computes the n-th term in the Fibonacci sequence
    // it uses recursion, which is slow. But we meant to be slow here.
    func fib(n int) (r int) {
    	if n == 1 || n == 2 {
    		return 1
    	} else {
    		return fib(n-1) + fib(n-2)
    	}
    }
    
    func spinner() {
    	spinner := `-\|/`
    	for {
    		for _, c := range spinner {
    			fmt.Printf("\r%c", c)
    			time.Sleep(100 * time.Millisecond)
    		}
    	}
    }
    
    func main() {
    	go spinner()
    	fmt.Printf("\r%v\n", fib(45))
    }
    
    ```
    




Goroutine 调用的函数执行结束（返回或达到结尾）时，该 goroutine 终止。`main` goroutine 终止时，其他所有 goroutine 也随之终止。除此之外，没有任何编程上的方法可以令一个 goroutine 结束另一个。

# 2 Channel

**Channel**（通道）是语言级别的 goroutine 间通信机制，允许在 goroutine 间收发值。 每个 channel 是一个特定类型的值的管道。



**类型**

Channel 类型 `chan T` 表示一个 `T` 类型的channel，只允许通过该channel传递 `T` 类型的值。Channel 的缓冲容量信息不在其类型信息中，而是存储在其底层数据结构中，是其值的一部分。

Channel 是引用类型，每个 channel 变量是其底层数据结构的引用。channel 的零值为 `nil`。

---

两个同类型的 channel 可以用 `==` 比较，若它们指向同一个底层 channel 则相等。Channel 也可与 `nil` 比较用来检测自身是否为 `nil`。





**声明**

直接声明变量会进行零值初始化：

```go
var ch chan int // nil
var ch [3]chan int // {nil, nil, nil}
```

使用 `make` 创建 channel：

```go
ch := make(chan int)       // unbuffered channel
ch := make(chan int, 0)    // unbuffered channel
ch := make(chan int, 3)    // buffered channel, capacity 3
```





**基本操作**

Channel 有两个基本操作：**发送**和**接收**

```go
ch <- x    // send x to ch
y := <-ch  // receive from ch
```

接收操作返回两个值：channel 中的值、表示是否成功接收的 bool 值：

```go
x, ok := <-ch
```





**关闭 channel**

内置函数 `close` 用以关闭 channel：

```go
close(ch)
```

关闭一个 channel `ch` 只是将其标记为已关闭，表示不会再有额外的通信经过 `ch`。关闭后的 channel 中依然还有可能缓冲着一些值。

- 往关闭的 channel `ch` 中发送任何值都会导致 panic。
- 从 `ch` 中接收会逐一获取 `ch` 缓冲着的值。若缓冲是空的，接收操作会返回对应类型的零值，以及 `false` 表示没有成功接收到值。
- 关闭一个已经关闭的 channel 会导致 panic。




`for-range`

`for-range` 可以作用于管道：

```go
for v := range ch { /* ... */ }
```

上述循环会在 `ch` 关闭时退出。它等价于：

```go
for {
	v, ok := <-ch
	if !ok {
		break
	}
	/* ... */
}
```

有了上述等价代码，我们可以归纳出下列表现：

- 如果 `ch` 没有关闭，则 `for-range` 会阻塞等待。
- 如果 `ch` 为 `nil`，则永久阻塞。




`nil` Channel

若 channel `ch` 为 `nil`，则向其发送值或从中接收值都会导致永久阻塞。这一特性在 `select` 语句中可以用来永久禁用其中某一个 case。`select` 语句会忽略相应的 case。

关闭 `nil` channel 会导致 panic。



## 2.1 无缓冲 Channel

**无缓冲 channel**（unbuffered channel）也被称作**同步 channel**（synchronous channel）。

一个 goroutine 向无缓冲 channel 发送一个值后，必须**等待**该值被其他 goroutine 接收。

一个 goroutine 试图从无缓冲 channel 接收值必须**等待**其他 goroutine 往该 channel 中发送一个值。

上述表现可以使发送和接收两个 goroutine 同步。



**示例：使用无缓冲 channel 同步两个 goroutine**

- 例 1
    
    ```go
    func main() {
    	// compute fib(40) + fib(45) using two goroutines
    	// sync them using a simple unbuffered channel
    	// just for demonstration
    
    	start := time.Now()
    
    	done := make(chan int)
    	go func() {
    		done <- fib(45)
    	}()
    
    	sum := fib(44) + <-done
    
    	duration := time.Since(start)
    
    	fmt.Println(sum)
    	fmt.Printf("%v\n", duration)
    
    	start = time.Now()
    	fmt.Println(fib(44) + fib(45))
    	fmt.Printf("%v\n", time.Since(start))
    }
    
    func fib(n int) (r int) {
    	if n == 1 || n == 2 {
    		return 1
    	} else {
    		return fib(n-1) + fib(n-2)
    	}
    }
    ```
    
- 例 2
    
    ```go
    done := make(chan struct{})
    go func() {
    	io.Copy(os.Stdout, conn) // NOTE: ignoring errors
    	log.Println("done")
    	done <- struct{}{}
    	// signal the main goroutine
    }()
    mustCopy(conn, os.Stdin)
    conn.Close()
    <-done // wait for background goroutine to finish
    ```
    


## 2.2 管道和单向 Channel

可以将 channel 首尾相连以将数个 goroutine 串为管道——即将一个的输出作为下一个的输入。我们以下列简单的教学例子来说明：

![image.png](/assets/images/260318-go-concurrency/image%203.png)

- Code
    
    ```go
    func main() {
    	cToS := make(chan int)
    	sToP := make(chan int)
    	N := 10
    
    	go counter(N, cToS)
    	go squarer(N, cToS, sToP)
    	printer(N, sToP)
    
    }
    
    func counter(N int, cToS chan<- int) {
    	for i := range N {
    		cToS <- i
    	}
    }
    
    func squarer(N int, cToS <-chan int, sToP chan<- int) {
    	for range N {
    		x := <-cToS
    		sToP <- x * x
    	}
    }
    
    func printer(N int, sToP <-chan int) {
    	for range N {
    		fmt.Println(<-sToP)
    	}
    }
    ```
    

在上述例子中三个函数事先约定好了总共有多少元素会通过管道。而在元素数未知时，可以使用 `for range` 循环：

- Code
    
    ```go
    func counter(N int, cToS chan<- int) {
    	for i := range N {
    		cToS <- i
    	}
    	close(cToS)
    }
    
    func squarer(cToS <-chan int, sToP chan<- int) {
    	for x := range cToS {
    		sToP <- x * x
    	}
    	close(sToP)
    }
    
    func printer(sToP <-chan int) {
    	for x := range sToP {
    		fmt.Println(x)
    	}
    }
    ```
    

上述例子中还使用了**单向 channel**（unidirectional channel）：作为函数参数的通道类型可以指定为**只接收**（`<-chan`）或**只发送**（`chan<-`）：

```go
func squarer(cToS <-chan int, sToP chan<- int)
```

- 单向特性是在编译时检查的，若违反，则报编译时错误。
- 单向 channel 是区别于双向 channel 的类型。

## 2.3 有缓冲 Channel

**有缓冲 channel**（buffered channel）维护一个元素队列。创建有缓冲 channel 时提供大于 0 的容量参数给 `make`：

```go
make(chan int, 3)
```

因为存在缓冲队列，有缓冲 channel 在容量未满时不会阻塞。在 channel 已满时再往其中发送值会导致和无缓冲 channel 一样的阻塞，此时发送方 goroutine 会等待直到有其他 goroutine 从 channel 中接收值。接收时从缓冲队列头部取出值，并将阻塞的值追加到队列尾部。

- 同样的，从空的有缓冲 channel 中取出值时会导致阻塞。

`len` 返回缓冲队列中有多少值，`cap` 返回缓冲队列的容量。在无缓冲 channel 上调用这两个函数总返回 0。

有缓冲 channel 可以用于处理对于服务器的并行请求：

```go
func mirroredQuery() string {
	responses := make(chan string, 3)
	go func() { responses <- request("asia.gopl.io") }()
	go func() { responses <- request("europe.gopl.io") }()
	go func() { responses <- request("americas.gopl.io") }()
	return <-responses // return the quickest response
}
func request(hostname string) (response string) { /* ... */ }
```

上例中如果使用无缓冲 channel，则较慢的两个 goroutine 会阻塞。这种现象称作 **goroutine 泄漏**（leak）。垃圾回收机制不会自动回收 goroutine。

## 2.4 选择有缓冲和无缓冲 channel

无缓冲 channel 提供更强的同步保证。

强同步在某些场景下会影响性能——无缓冲 channel 必须等待发送的值被接收，相应的 goroutine 也随之等待。若使用有缓冲 channel，则 goroutine 将值发送后可以继续执行，无需等待（假设容量足够）。

在管道架构中，如果上游产生值的速度快过下游处理的速度，则其间的缓冲队列会长时间处于满的状态，导致大量阻塞；相反，如果下游处理的速度快过上游产生的速度，则缓冲队列会长时间处于空的状态，也会导致大量阻塞。此时缓冲队列的意义不大。这个问题可以通过将速度较慢的部分采用并行处理：即创建新的 goroutine 并行处理同一部份工作。

# 3 `select` 多路复用

本节用一个计时器例子介绍 `select` 关键字。

以一个简单的倒数程序开始：

```go
func main() {
	fmt.Println("Commencing countdown.")
	tick := time.Tick(1 * time.Second)
	for countdown := 10; countdown > 0; countdown-- {
		fmt.Println(countdown)
		<-tick
	}
	launch()
}
```

- `time.Tick` 返回一个 `<-chan time.Time` 类型的 channel，并启动一个后台 goroutine 定期向该通道发送时间。
    
    ```go
    func Tick(d Duration) <-chan Time
    ```
    
    - `d` 为发送间隔

接下来我们添加按键中断倒数过程的功能：

```go
abort := make(chan struct{})
go func() {
	os.Stdin.Read(make([]byte, 1))
	abort <- struct{}{}
}()
```

每次倒数循环的迭代都必须选择「中断」和「正常」两种行为之一（通过从 channel 中接收值），但这二者不能顺序执行因为接收操作会阻塞 goroutine。因此这些操作需要通过 `select` 语句进行多路复用：

```go
select {
case <channel-operation1>:
	// ...
case <channel-operation2>:
	// ...
// ...
default:
	// ...
}
```

`select` 会阻塞当前 goroutine 直到其其中一个 case 准备就绪。

- `select{}` 永久阻塞。
- 多个 case 同时就绪时，`select` 从中任选一个。所有 case 有等概率被选中。
- `default` 分支也就是所有 case 都未就绪时 `select` 的行为。
- 在 `select` 语境下，`nil` 通道可以用来阻塞某一个 case。

使用 `select` 修正我们的倒数程序：

```go
func main() {

	abort := make(chan struct{})
	go func() {
		os.Stdin.Read(make([]byte, 1))
		abort <- struct{}{}
	}()

	fmt.Println("Commencing countdown.")

	tick := time.Tick(time.Second)
	for countdown := 10; countdown > 0; countdown-- {
		fmt.Println(countdown)
		select {
		case <-tick:
		case <-abort:
			fmt.Println("FUCK IS ABORTED!")
			return
		}
	}

	launch()
}
```

`Tick` 容易导致 goroutine 泄漏——倒数中断时并没有停止 ticker 所在的 goroutine。因此一般只推荐在需要持续 tick 且无需控制停止（即生命期就是应用程序的全生命周期）时使用。其他情况下建议使用 `time.NewTicker`：

```go
ticker := time.NewTicker(time.Second)
<-ticker.C
ticker.Stop()
```

# 4 `sync` 包

## 4.1 `sync.WaitGroup` 多协程调用

上述的 `spinner` 例子中，主程序创建 goroutine 显示 spinner 效果后调用了 `fib(45)` 并等待计算完毕。`fib(45)` 返回后主程序触底结束。

很多时候我们需要等待一组 goroutine 全部完成再继续主程序的执行。`sync` 包就提供了一个并发安全的计数器 `sync.WaitGroup`，用以等待多个 goroutine 完成：每创建一个 goroutine 令计数器递增，每一个 goroutine 接收令计数器递减，主程序中则用 `Wait()` 阻塞，直到计数器变为 0。例如：

```go
func worker(id int, wg *sync.WaitGroup) {
    // 确保函数退出时通知 WaitGroup
    defer wg.Done()
    
    fmt.Printf("Worker %d 开始工作\n", id)
    time.Sleep(time.Second) // 模拟工作
    fmt.Printf("Worker %d 完成工作\n", id)
}

func main() {
    var wg sync.WaitGroup
    
    // 启动 3 个 worker
    for i := 1; i <= 3; i++ {
        wg.Add(1) // 增加计数器
        go worker(i, &wg)
    }
    
    // 等待所有 worker 完成
    wg.Wait()
    fmt.Println("所有工作已完成，主程序退出")
}
```

- `Add(delta int)`
- `Done()`：递减计数器，等价于 `Add(-1)`
- `Wait()`：阻塞直到计数器为 0。

要点：

- `WaitGroup` 必须通过指针传递，否则会复制出新的计数器，原计数器无法正确递增递减，造成死锁。
- `Done` 常用 `defer` 确保 `panic` 时也能正确递减计数。

## 4.2 `sync.Once` 懒初始化

**懒初始化**（lazy initialization）指的是将一系列实体的初始化操作推迟到恰好在实际需要使用这些实体之前。例如加载配置文件，如果程序一开始就加载配置，却迟迟不使用，则即浪费了内存，又延长程序的加载时间。`sync.Once` 可以实现懒初始化，先进行声明，而后在实际用到资源时才进行实际的加载。

```go
type Config struct{ /* ... */ }
var instance *Config
var once sync.Once

func InitConfig() *Config {
	once.Do(func(){
		instance = &Config{ /* ... */ }
	})
	return instance
}
```

- 只有在第一次调用 `InitConfig` 是才会执行 `once.Do` 中的函数。

## 4.3 `sync.Mutex` `sync.RWMutex` 锁

`sync.Mutex` 实现了互斥锁。

```go
var mu sync.Mutex
```

```go
func (m *Mutex) Lock()
func (m *Mutex) Unlock()
```

使用示例：

```go
func Deposit(amount int) {
	mu.Lock()
	balance = balance + amount
	mu.Unlock()
}
```

可以使用 `defer` 来确保锁的释放，例如：

```go
func Balance() int {
	mu.Lock()
	defer mu.Unlock()
	return balance
}
```

使用互斥锁会令所有操作顺序地获取和释放锁，包括读取和写入操作。而多个并发的读取操作实际上无需互斥锁保护。在读取操作远多于写入操作的情景下，我们可以使用允许多个并行读取，但总只允许单个写入的读写互斥锁 `sync.RWMutex`：

```go
var mu sync.RWMutex
var balance int
func Balance() int {
	mu.RLock()
	defer mu.RUnlock()
	return balance
}
```

读写互斥锁由共享的**读者锁**（readers lock）以及互斥的**写者锁**（writer lock）组成。

- `mu.RLock` 和 `mu.RUnlock` 控制读者锁
- `mu.Lock` 和 `mu.Unlock` 控制写者锁

切记使用读者锁的临界区不可出现任何写入操作。

## 4.4 `sync.Map` 并发安全的映射

内置类型 `map` 不是并发安全的——并发写操作会导致 fatal error。解决方法是使用额外的互斥锁保护写操作，或使用 `sync` 包提供的并发安全的 `sync.Map`：

```go
func main() {
	var m sync.Map

	// store
	m.Store("name", "dmc")
	m.Store("age", 99)

	// query
	age, ok := m.Load("age")
	fmt.Println(age, ok)

	// traversal
	m.Range(func(key, value any) bool {
		fmt.Println(key, value)
		return true
	})

	// delete
	m.Delete("age")
	age, ok = m.Load("age")
	fmt.Println(age, ok)

	// load if exists, store otherwise
	name, ok := m.LoadOrStore("name", "dmcc")
	fmt.Println(name, ok)
}
```

- `sync.Map` 是不限定类型的键值存储，`Store` 接收的、`Load` 返回的值类型均为 `any`。
- `sync.Map` 为了并发安全牺牲了性能。

## 4.5 `sync/atomic` 原子操作

下列语句不是原子操作：

```go
i = i + 2
```

在 CPU 层面，这一语句包括三个步骤：读取 `i` 的值到寄存器，执行加法，将结果写入内存。在上述过程中若发生上下文切换则可能出现并发危险。当然可以用互斥锁保护：

```go
mu.Lock()
defer mu.Unlock()
i = i + 2
```

但也可以使用原子的算术操作，这些操作由 `sync/atomic` 包提供：

```go
func AddInt64(addr *int64, delta int64) int64
func StoreInt64(addr *int64, val int64)
func LoadInt64(addr *int64) int64
func SwapInt64(addr *int64, newVal int64) int64
func CompareAndSwapInt64(addr *int64, oldVal, newVal int64) bool
```

以上五个函数除了对 `int64` 类型定义了之外，还给 `int32` `uint32` `uint64` `uintptr` 类型有定义。如 `AddInt32` `LoadUint64` 等。

```go
atomic.AddInt32(&sum, int32(42))
```

`atomic.Value` 支持对任何接口类型的原子操作 `Load` `Store` `Swap` `CompareAndSwap`。

## 4.6 `sync.Pool` 临时对象池

`sync.Pool` 是一个临时对象池，用于存储和复用临时对象以减轻 GC 的压力。

- 避免短时间内重复创建相同的对象，减少 GC 压力。尤其在高并发场景下，每个 goroutine 都在频繁地创建大对象时。
- 并非持久化存储。不适合存储带状态的对象如数据库连接。
- `sync.Pool` 本身为并发安全的

`New()` `Get()` `Put()`

- `New()`：池对象构造函数，当 `Get()` 时池为空则调用
- `Get()`：从池中获取对象，可能返回 `nil` 或新构造的对象
- `Put()`：将用完的对象放回池中复用

```go
pool := sync.Pool{
    New: func() interface{} { return 0 }, // 1. 定义构造函数
}
value := pool.Get().(int)  // 2. 获取对象（为空则New）
pool.Put(value)            // 3. 放回复用
```

# 5 `Context` 接口

`context.Context` 是一个用于携带截止时间、取消信号和请求作用域值的接口：

```go
type Context interface {
    // 返回此 Context 是否应该被取消
    Done() <-chan struct{}
    
    // 如果 Done 未关闭，返回 nil；如果 Done 已关闭，返回取消原因
    Err() error
    
    // 返回此 Context 应该被取消的时间
    Deadline() (deadline time.Time, ok bool)
    
    // 获取 key 对应的值
    Value(key interface{}) interface{}
}
```

- `Deadline` 设置  `Context` 被取消的时间，即截止时间。
- `Done` 返回一个只读 channel。若 Context 被取消或超时，该 channel 则关闭，表示 Context 的链路结束。
- `Err` 返回 `Context` 结束的原因。
    - `Context` 被取消 → context canceled
    - `Context` 超时 → context deadline exceeded
- `Value` 从 `Context` 中获取键对应的值。

`Context` 主要用于在父子 goroutine 间进行值传递和收发 cancel 信号：

- 并发控制，控制 goroutine 优雅地退出
- 上下文信息传递

## 5.1 创建 Context

首先使用 `context.Background()` 或 `context.TODO()` 创建**根** `Context`：

```go
// 返回一个空的 Context，永远不会被取消，没有截止时间，没有值
ctx1 := context.Background()

// 返回一个空的 Context，通常用于还未确定用哪种 Context 的情况
ctx2 := context.TODO()
```

根 `Context` 不具备任何功能，需要用下列函数进行派生：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

![image.png](/assets/images/260318-go-concurrency/image%204.png)

- 例子：`context.WithCancel`
    
    `WithCancel` 返回一个 `Context` 和一个 `CancelFunc`。将返回的 `Context` 传递到新的 goroutine 中可以控制这些 goroutine 的关闭。一旦调用返回的取消函数，则对应的 `Context` 以及由其再派生的子 `Context` 都会同步收到信号（通过 `ctx.Done` 获取）：
    
    ```go
    func main() {
    	parent := context.Background()
    
    	ctx, cancel := context.WithCancel(parent)
    
    	go func() {
    		for {
    			select {
    			case <-ctx.Done():
    				fmt.Println("Goroutine canceled.")
    				return
    			default:
    				fmt.Println("working...")
    				time.Sleep(500 * time.Millisecond)
    			}
    		}
    	}()
    
    	time.Sleep(2 * time.Second)
    	fmt.Println("cancel signal sent...")
    	cancel()
    
    	time.Sleep(1 * time.Second)
    }
    ```
    
- 例子：`context.WithDeadLine` 与 `context.WithTimeout`
    
    `WithDeadline` 接收父 `Context` 以及一个表示截止时间的时间戳 `d time.Time`，同样返回子 `Context` 和 `CancelFunc`。截止时间到后，Context 会因超时而结束。也可以在超时前调用取消函数来手动取消：
    
    ```go
    func main() {
    	parent := context.Background()
    
    	ctx, cancel := context.WithDeadline(parent, time.Now().Add(3*time.Second))
    	defer cancel()
    
    	var wg *sync.WaitGroup = new(sync.WaitGroup)
    	wg.Add(1)
    	go func(wg *sync.WaitGroup) {
    		for {
    			select {
    			case <-ctx.Done():
    				fmt.Println("Goroutine canceled.")
    				fmt.Println(ctx.Err())
    				wg.Done()
    				return
    			default:
    				fmt.Println("working...")
    				time.Sleep(500 * time.Millisecond)
    			}
    		}
    	}(wg)
    	wg.Wait()
    }
    ```
    
    `WithTimeout` 与 `WithDeadline` 类似，不过第二参数为 `time.Duration`。
    

`context.WithValue` 派生一个子 `Context` 用以传递值，一般用于上下文信息的传递。

```go
func main() {
	ctx := context.WithValue(context.Background(), "request_id", "R001")
	go func() {
		fmt.Printf("The request_id is %v\n", ctx.Value("request_id"))
	}()
	time.Sleep(time.Second)
}
```

## 5.2 使用 Context 的并发控制

考虑以下 RPC（remote procedure cal） 链：原始请求调用 RPC1，RPC1 中又需要调用 RPC2，如此往复形成一条调用链。在全过程中如果 RPC1 中出错，我们希望能直接取消所有后续 RPC，返回失败。如果使用 Context，我们可以直接通知子 goroutine 关闭，不必等待其进行无意义的计算和 I/O。

![image.png](/assets/images/260318-go-concurrency/image%205.png)

# 6 `time.Timer` 与 `time.Ticker` 定时器

## 6.1 `Timer`

`time.Timer` 是一个定时器，代表未来的一个单次事件。

```go
type Timer struct {
	C <-chan Time
	r runtimeTimer
}

func NewTimer(d Duration) *Timer
func (t *Timer) Stop() bool
func (t *Timer) Reset(d Duration) bool
```

设定时间过后，`Timer` 会向自身的 channel `C` 中写入一个系统时间，以触发事件：

```go
func main() {
	timer := time.NewTimer(2 * time.Second)
	fmt.Println("Timer fired at", <-timer.C)
}
```

`Stop` 可以用来停止定时器。如果在定时器触发前停止则返回 `true`，否则返回 `false`。

```go
func main() {
	timer := time.NewTimer(time.Second)
	fmt.Println(timer.Stop())
	time.Sleep(time.Second)
	fmt.Println(timer.Stop())
}
// will print
// true
// false
```

`Reset` 可以重新激活过期或被停止的 `Timer`。如果 `Timer` 处于激活状态下重置，`Reset` 返回 `true`；如果 `Timer` 已经过期或被停止，则返回 `false`。

## 6.2 `time.After` `time.AfterFunc`

`After` 接受一个 `Duration` `d`，返回一个只读 `<-chan Time`。在 `Duration` 过后，这个 channel 会接收到当前时间。

```go
func After(d Duration) <-chan Time {
	return NewTimer(d).C
}
```

`AfterFunc` 接收一个 `Duration` `d` 和一个函数 `f`，在 `Duration` 过后在新的 goroutine 中**异步执行**该函数。

```go
func AferFunc(d Duration, f func()) *Timer
```

`AfterFunc` 的返回值是一个 `*Timer`，可以用来取消或调整（`Stop` `Reset`）这个定时任务。

## 6.3 `Ticker`

`Ticker` 是一个周期性定时器，定期向自身的 channel `C` 中写入当前系统时间。

```go
type Ticker struct {
	C <-chan Time
	r runtimeTimer
}
```

使用 `NewTicker` 创建：

```go
func NewTicker(d Duration) *Ticker
```

- 例子：不断读秒，按任意键停止
    
    ```go
    func main() {
    	ticker := time.NewTicker(time.Second)
    
    	ch := make(chan struct{})
    	defer close(ch)
    
    	go func(ticker *time.Ticker) {
    		defer ticker.Stop()
    		i := 0
    		for {
    			select {
    			case <-ticker.C:
    				i++
    				fmt.Println(i)
    			case <-ch:
    				fmt.Println("ticker stopped.")
    				return
    			}
    		}
    	}(ticker)
    
    	var input string
    	fmt.Scanln(&input)
    	ch <- struct{}{}
    }
    ```
    

使用 `Stop` 停止 `Ticker`

```go
ticker.Stop()
```

# 7 协程池

Goroutine 比 OS 线程轻量得多，但过多的并发 goroutine 会对调度系统、垃圾回收和系统内存都造成巨大压力——太高的并发数会令系统性能不升反降。

协程**池**（pool）可以协助将协程控制在一定的数量。协程池的核心思想在于创建固定数量的 worker goroutine，通过 channel 分发任务 `Task`。

首先定义任务和池：

```go
type Task struct {
	f func() error
}
type Pool struct {
	RunningWorkers int64   // current number of running  workers
	Capacity int64         // max number of workers
	JobCh chan *Task       // task queue
	sync.Mutex
}
```

Worker 可以如下不断从任务队列（一个 channel）中取任务执行，例如：

```go
p := NewPool(capacity, taskNum)
// capacity is the max number of workers
// taskNum is the buffer size of JobCh
for task := range p.JobCh {
	// ...
}
```

## 7.1 方法定义

```go
func NewPool(capacity int, taskNum int) *Pool
func NewTask(funcArg func() error) *Task

func (p *Pool) AddTask(task *Task)
func (p *Pool) Run()
```

## 7.2 使用流程

![image.png](/assets/images/260318-go-concurrency/image%206.png)

## 7.3 实现示例

```go
type Task struct {
	f func() error
}

func NewTask(funcArg func() error) *Task {
	return &Task{
		f: funcArg,
	}
}

type Pool struct {
	RunningWorkers int64
	Capacity       int64
	JobCh          chan *Task
	sync.Mutex
}

func NewPool(capacity int64, taskNum int64) *Pool {
	return &Pool{
		Capacity: capacity,
		JobCh:    make(chan *Task, taskNum),
	}
}

func (p *Pool) Cap() int64 {
	return p.Capacity
}

func (p *Pool) incRunning() {
	atomic.AddInt64(&p.RunningWorkers, 1)
}

func (p *Pool) decRunning() {
	atomic.AddInt64(&p.RunningWorkers, -1)
}

func (p *Pool) GetRunningWorkers() int64 {
	return atomic.LoadInt64(&p.RunningWorkers)
}

func (p *Pool) run() {
	p.incRunning()
	go func() {
		defer func() {
			p.decRunning()
		}()
		for task := range p.JobCh {
			task.f()
		}
	}()
}

func (p *Pool) AddTask(task *Task) {
	p.Lock()
	defer p.Unlock()

	if p.GetRunningWorkers() < p.Cap() {
		p.run()
	}

	p.JobCh <- task
}

func main() {
	pool := NewPool(3, 10)

	for i := range 20 {
		pool.AddTask(NewTask(func() error {
			fmt.Printf("Task num %v\n", i)
			return nil
		}))
	}

	time.Sleep(time.Second)
}

```