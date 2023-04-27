# Channel
Go 语言采用了 CSP 并发编程思想，将通道作为协程之间交流的原语，屏蔽了传统多线程编程中底层实现的诸多细节。<br />通道不仅有形象的语法，而且产生了很多经典的富有表现力的并发模型（如`ping-pong`、`fin-in`、`fin-out`、`pipeline`）。<br />通道的实现底层是用**锁**实现的环形队列，在读取和写入时如果不能直接操作则会被放入等待队列中陷入休眠状态。<br />借助 Go 运行时的调度器，通道不会堵塞程序的执行，并且协程能够在需要时被快速唤醒。<br />在实际中，一个协程时常会处理多个通道，当然不希望由于一个通道的读写陷入阻塞，影响其他通道的正常读写。因此在实际中，更多会使用`select`多路复用的机制同时监听多个通道是否准备就绪。`select`在底层会锁住所有的通道并采取**随机**的方式保证公平地遍历所有通道。
<a name="F6cSj"></a>
# 底层原理
Go 语言中经常被人提及的一个设计模式：**不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存**。Goroutine 之间通过 channel 传递数据，channel 作为 Go 语言的核心数据结构是支撑 Go 语言高性能并发编程模型的重要结构。<br />channel 在 Runtime 内部表示是`[runtime.hchan](https://github.com/golang/go/blob/master/src/runtime/chan.go#L33)`，该结构体中包含了用于保护成员变量的互斥锁，从某种程度上说，channel 是一个用于**同步**和**通信**的有锁队列。<br />`hchan 结构体源码：`
```go
type hchan struct {
    qcount    	uint        	
    dataqsiz  	uint        	
    buf      	unsafe.Pointer  
    elemsize  	uint16      	
    closed    	uint32      	
    elemtype  	*_type      	
    sendx    	uint        	
    recvx    	uint        	
    recvq    	waitq        	
    sendq    	waitq        	

    lock    	mutex        	
}

type waitq struct {        	// 双向链表
    first  	*sudog
    last  	*sudog
}
```
`waitq`中连接的是一个`sudog`双向链表，保存的是等待中的 Goroutine。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669861858504-8160403f-b7b0-4f3c-8ec7-716ba0cc4429.png#averageHue=%23f8f2ec&clientId=u1f6884a7-cc12-4&from=paste&id=uaf40e0d4&originHeight=490&originWidth=1033&originalType=binary&ratio=1&rotation=0&showTitle=false&size=48617&status=done&style=none&taskId=u1686dece-1364-4caa-8fd5-4b8c8acda88&title=)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669953583388-2884a466-5263-4341-84f3-10e0dbde8dcf.png#averageHue=%23f8f3f0&clientId=u202cbce1-4dc2-4&from=paste&id=u2f240014&originHeight=528&originWidth=932&originalType=binary&ratio=1&rotation=0&showTitle=false&size=79597&status=done&style=none&taskId=u5574b3e7-64b5-4596-a970-65eb9199aa3&title=)<br />对于有缓存的通道，存储在`buf`中的数据虽然是线性数组，但是用数组和序号`recvx`、`recvq`模拟了一个环形队列，`recvx`到`sendx`的距离代表通道队列中的元素数量

- `recvx`可以找到从`buf`哪个位置获取通道中的元素，
- `sendx`能够找到写入时放入`buf`的位置，

这样做主要是为了**重用**已经使用过的空间，当到达循环队列的末尾时，`sendx`会置为 0，以防止其下一次写入 0 号位置，开始循环利用空间。<br />这同样意味着，当前的通道中只能放入指定大小的数据。当通道中的数据满了后，再次写入数据将陷入等待，直到第 0 号位置被取出后，才能继续写入。<br />`sudog`代表着等待队列中的一个 Goroutine，G 与同步对象（指 channel）关系是多对多的。

- 一个 G 可以出现在许多等待队列上，因此一个 G 可能有多个 sudog。
- 多个 G 可能正在等待同一个同步对象，因此一个对象可能有许多 `sudog`。

`sudog` 是从特殊池中分配出来的。使用 `acquireSudog` 和 `releaseSudog` 分配和释放它们。
<a name="EkWVe"></a>
# 创建 chan
声明通道：`var name chan T`，未初始化的通道是`nil`

- `name`代表`chan`的名字，为用户自定义的；
- `chan T`代表通道的类型，
- `T`代表通道中的元素类型。

在声明时，`channel`必须与一个实际的类型`T`绑定在一起，代表通道中能够读取和传递的元素类型。<br />通道的表示形式有三种：

1. `chan T`：可读写通道（可读写通道，可以转为单向通道，将单通道赋予给可读写通道，在编译时会报错）
2. `chan<-T`：只写通道
3. `<-chan T`：只读通道

一个**未初始化**的通道在编译时和运行时并不会报错，不过，显然无法向通道中写入或读取任何数据。要对通道进行操作，需要使用`make`操作符，`make`会初始化通道，在内存中分配通道的空间。<br />`make(chan int, 3)`会调用到`[runtime.makechan](https://github.com/golang/go/blob/master/src/runtime/chan.go#L72)`函数。根据 channel 中收发元素的类型和缓冲区的大小初始化`runtime.hchan`和缓冲区。

- 第 1 个参数代表通道的**类型**，
- 第 2 个参数代表通道中元素的**大小**。

`makechan()`会判断元素的大小、对齐等。最重要的是，它会在内存中分配元素大小。
```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem
	...
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
	...
	return c
}
```

- 若缓冲区所需大小为 0，就只会为 hchan 分配一段内存；
- 若缓冲区所需大小不为 0 且 elem 不包含指针，会为 hchan 和 buf 分配一块连续的内存；
- 若缓冲区所需大小不为 0 且 elem 包含指针，会单独为 hchan 和 buf 分配内存，因为当元素中包含指针时，需要单独分配空间才能正常进行垃圾回收。
<a name="P5ATP"></a>
# chan 写入数据
发送数据到 channel，`ch <- data`会调用到`[runtime.chansend()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L160)`函数中，该函数包含了发送数据的全部逻辑。<br />`block`表示当前的发送操作是否是**阻塞**调用。如果 channel 为`nil`：

1. 对于非阻塞的发送，直接返回 `false`，
2. 对于阻塞的发送，将 goroutine 挂起，并且**永远不会返回**。

对 channel 加锁，防止多个线程并发修改数据，如果 channel 已关闭，报错并中止程序。<br />`runtime.chansend()`函数的执行过程可以分为以下三个部分：

1. **直接发送**：当存在等待的接收者时，通过直接将数据发送给阻塞的接收者；
2. **发送到缓冲区**：当缓冲区存在空余空间时，将发送的数据写入缓冲区；
3. **阻塞发送**：当不存在缓冲区或缓冲区已满时，等待其他 Goroutine 从 channel 接收数据。
> 对于无缓冲通道，能够向通道写入数据的前提是必须有另一个协程在读取通道。否则，当前的协程会陷入**休眠**状态，直到能够向通道中成功写入数据。
> 无缓冲通道的读与写应该位于不同的协程中，否则，程序将陷入**死锁**的状态。

发送数据整个流程大致如下：

![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669866136492-575d9604-a182-49a1-9013-c35ae38cf661.png#averageHue=%23f9f7f4&clientId=ued5c93b2-ec35-4&from=paste&id=dvdF1&originHeight=396&originWidth=972&originalType=binary&ratio=1&rotation=0&showTitle=false&size=35366&status=done&style=none&taskId=ue88e6a5e-40c0-42b8-9660-9561ec3c18a&title=)<br />注意，发送数据的过程中包含几个会触发 Goroutine 调度的时机：

- 发送数据时发现从 channel 上存在等待接收数据的 Goroutine，立刻设置处理器的`runnext`属性，但是并不会立刻触发调度；
- 发送数据时并没有找到接收方并且缓冲区已经满了，这时会将自己加入 channel 的`sendq`队列并调用`gopark()`触发 Goroutine 的调度让出 M 的使用权。
<a name="k8Z3s"></a>
## 直接发送
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680859306706-fc45953e-6871-498a-a2b5-7ca04ef157fd.png#averageHue=%23f9f9f7&clientId=u4b0d26b9-76ce-4&from=paste&height=273&id=ub18e4dd1&originHeight=546&originWidth=1442&originalType=binary&ratio=2&rotation=0&showTitle=false&size=263820&status=done&style=none&taskId=u0469df1c-8fe4-481f-bee4-bc52afaa730&title=&width=721)<br />如果目标 channel 没有被关闭且`recvq`队列中已经有处于读等待的 Goroutine，那么`runtime.chansend()`会从接收队列`recvq`中取出最先陷入等待的 Goroutine 并调用`[runtime.send()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L294)`函数直接向它发送数据，因为有接收者在等待，所以如果有缓冲区，那么此时缓冲区也一定是空的。
```go

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
  ...
  if sg.elem != nil {
    // 直接把要发送的数据copy到接收者的栈空间
    sendDirect(c.elemtype, sg, ep)
    sg.elem = nil
  }
  gp := sg.g
  unlockf()
  gp.param = unsafe.Pointer(sg)
  if sg.releasetime != 0 {
    sg.releasetime = cputicks()
  }
  // 设置对应的goroutine为可运行状态
  goready(gp, skip+1)
}
```

- `[sendDirect()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L335)`函数调用`[memmove()](https://github.com/golang/go/blob/master/src/runtime/stubs.go#L109)`函数进行数据的内存拷贝。
- `[goready()](https://github.com/golang/go/blob/master/src/runtime/proc.go#L390)`函数将等待接收数据的 Goroutine 标记成可运行状态（`Grunnable`）并把该 Goroutine 发到发送方所在的处理器（P）的`runnext`上等待执行，该处理器在下一次调度时会立刻唤醒数据的接收方。注意，只是放到了`runnext`中，并没有立刻执行该 Goroutine。
<a name="Pif9K"></a>
## 发送到缓冲区
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680859346406-dc56471d-769d-4f7d-ab1a-583688c26724.png#averageHue=%23fdfdfc&clientId=u4b0d26b9-76ce-4&from=paste&height=342&id=u212ce97a&originHeight=684&originWidth=1420&originalType=binary&ratio=2&rotation=0&showTitle=false&size=281060&status=done&style=none&taskId=ueb4a5eea-c903-4416-9e02-ab0d9ddd9f2&title=&width=710)<br />如果缓冲区未满，则将数据写入缓冲区，找到缓冲区要填充数据的索引位置，调用`[typedmemmove](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go#L157)`函数将数据拷贝到缓冲区中，然后重新设值`sendx`偏移量。
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  ...
  // 如果缓冲区没有满，直接将要发送的数据复制到缓冲区
  if c.qcount < c.dataqsiz {
    // 找到 buf 要填充数据的索引位置
    qp := chanbuf(c, c.sendx)
    ...
    // 将数据拷贝到 buf 中
    typedmemmove(c.elemtype, qp, ep)
    // 数据索引前移，如果到了末尾，又从 0 开始
    c.sendx++
    if c.sendx == c.dataqsiz {
      c.sendx = 0
    }
    // 元素个数加 1，释放锁并返回
    c.qcount++
    unlock(&c.lock)
    return true
  }
  ...
}
```
<a name="KL4mk"></a>
## 阻塞发送
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680859379096-2b0c8ac2-3a33-4bd7-83b4-79daca3494f0.png#averageHue=%23ebf0d5&clientId=u4b0d26b9-76ce-4&from=paste&height=145&id=ub4e590c2&originHeight=290&originWidth=1408&originalType=binary&ratio=2&rotation=0&showTitle=false&size=218802&status=done&style=none&taskId=ub03f12e3-6d28-4141-b958-994e1922dee&title=&width=704)<br />当 channel 没有接收者能够处理数据时，向 channel 发送数据会被下游阻塞，使用`select`关键字可以向 channel 非阻塞地发送消息：
```go

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  ...
  // 缓冲区没有空间了，对于非阻塞调用直接返回
  if !block {
    unlock(&c.lock)
    return false
  }
  // 创建sudog对象
  gp := getg()
  mysg := acquireSudog()
  mysg.releasetime = 0
  if t0 != 0 {
    mysg.releasetime = -1
  }
  mysg.elem = ep
  mysg.waitlink = nil
  mysg.g = gp
  mysg.isSelect = false
  mysg.c = c
  gp.waiting = mysg
  gp.param = nil
  // 将sudog对象入队
  c.sendq.enqueue(mysg)
  // 进入等待状态
  gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
  ...
}
```

- 对于非阻塞的调用会直接返回，
- 对于阻塞的调用会创建`sudog`对象并将`sudog`对象加入发送等待队列。调用`gopark()`将当前 Goroutine 转入`Gwaiting`状态。调用`gopark()`之后，在使用者看来向该 channel 发送数据的代码语句会被阻塞。
<a name="LWHZt"></a>
# chan 读取数据
从 channel 获取数据最终调用到`[runtime.chanrecv()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L457)`函数。

- 当从一个`nil`的 channel 接收数据时，直接调用`gopark()`让出 M 的使用权。
- 如果当前 channel 已被关闭且缓冲区中没有数据，直接返回。

`runtime.chanrecv()`函数的具体执行过程可以分为以下三个部分：

- **直接接收**：当存在等待的发送者时，通过`[runtime.recv()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L615)`从阻塞的发送者或者缓冲区中获取数据；
- **从缓冲区接收**：当缓冲区存在数据时，从 channel 的缓冲区中接收数据；
- **阻塞接收**：当缓冲区中不存在数据时，等待其他 Goroutine 向 channel 发送数据。
> 和写入数据一样，如果不能直接读取通道的数据，那么当前的读取协程将陷入**阻塞**，直到有协程写入通道为止。
> 读取通道还有两种返回值的形式，借助编译时将该形式转换为不同的处理函数。
> 1. 第1个返回值仍然为通道读取到的数据，
> 2. 第2个返回值为布尔类型，返回值为`false`代表当前通道已经关闭。

注意，接收数据的过程中包含几个会触发 Goroutine 调度的时机：

- 当 channel 为 nil 时
- 当 channel 的缓冲区中不存在数据并且`sendq`中也不存在等待的发送者时
<a name="JWyxO"></a>
## 直接接收
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680859506867-be3d8c8a-ed3c-4f02-99ae-8430494e1917.png#averageHue=%23f9f9f7&clientId=u4b0d26b9-76ce-4&from=paste&height=275&id=uc0eff107&originHeight=550&originWidth=1440&originalType=binary&ratio=2&rotation=0&showTitle=false&size=250518&status=done&style=none&taskId=u71203167-1be6-40ce-bea5-6844ff8fce3&title=&width=720)<br />当 channel 的`sendq`队列中包含处于发送等待状态的 Goroutine 时，调用`[runtime.recv()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L615)`直接从这个发送者那里提取数据。因为有发送者在等待，所以如果有缓冲区，那么缓冲区一定是满的。<br />`runtime.recv()`函数会根据缓冲区的大小分别处理不同的情况：

- 如果 channel 不存在缓冲区：调用`[typedmemmove](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go#L157)`直接从发送者那里提取数据。
- 如果 channel 存在缓冲区：
   1. 调用`typedmemmove`将缓冲区中的数据拷贝到接收方的内存地址；
   2. 调用`typedmemmove`将发送者数据拷贝到缓冲区，并唤醒发送者。

无论发生哪种情况，Runtime 都会调用`goready()`将等待发送数据的 Goroutine 标记成可运行状态（`Grunnable`）并将当前处理器的`runnext`设置成发送数据的 Goroutine，在调度器下一次调度时将阻塞的发送方唤醒。
<a name="y21ay"></a>
## 从缓冲区接收
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680859527961-c2f12a60-08f4-41ba-a920-57aee67e6f14.png#averageHue=%23fdfdfb&clientId=u4b0d26b9-76ce-4&from=paste&height=517&id=u29770821&originHeight=1034&originWidth=1384&originalType=binary&ratio=2&rotation=0&showTitle=false&size=394970&status=done&style=none&taskId=u871de734-a131-4c2b-96d6-690c305f230&title=&width=692)<br />如果 channel 缓冲区中有数据且发送者队列中没有等待发送的 Goroutine 时，直接从缓冲区中`recvx`的索引位置取出数据：
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  ...
  // 如果缓冲区中有数据
  if c.qcount > 0 {
    qp := chanbuf(c, c.recvx)
    // 从缓冲区复制数据到ep
    if ep != nil {
      typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    // 接收数据的指针前移
    c.recvx++
    // 环形队列，如果到了末尾，再从 0 开始
    if c.recvx == c.dataqsiz {
      c.recvx = 0
    }
    // 缓冲区中现存数据减一
    c.qcount--
        unlock(&c.lock)
    return true, true
  }
  ...
}
```
<a name="A2F5f"></a>
## 阻塞接收
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680859558477-6e9a2725-2ebf-4706-9771-fa667cefd5c0.png#averageHue=%23eaefd4&clientId=u4b0d26b9-76ce-4&from=paste&height=146&id=u6383fcb9&originHeight=292&originWidth=1434&originalType=binary&ratio=2&rotation=0&showTitle=false&size=212123&status=done&style=none&taskId=u4e051f23-e237-4ec5-981a-29c58d0636c&title=&width=717)<br />当 channel 的发送队列中不存在等待的 Goroutine 并且缓冲区中也不存在任何数据时，从管道中接收数据的操作会被阻塞，使用`select`关键字可以非阻塞地接收消息：
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  ...
  // 非阻塞，直接返回
  if !block {
    unlock(&c.lock)
    return false, false
  } 
  // 创建 sudog
  gp := getg()
  mysg := acquireSudog()
  ···
  gp.waiting = mysg
  mysg.g = gp
  mysg.isSelect = false
  mysg.c = c
  gp.param = nil
  // 将 sudog 添加到等待接收队列中
  c.recvq.enqueue(mysg)
  // 阻塞 Goroutine，等待被唤醒
  gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
  ...
}
```

- 如果是非阻塞调用，直接返回。
- 阻塞调用会将当前 Goroutine 封装成`sudog`，然后将`sudog`添加到等待接收队列中，调用`gopark()`让出 M 的使用权并等待调度器的调度。
<a name="y5T15"></a>
# 关闭 chan

- 在正常读取的情况下，通道返回的`ok`为`true`。
- 通道在关闭时仍然会返回，但是`data`为其类型的零值，`ok`也变为了`false`。

和通道读取不同的是，不能向已经关闭的通道中写入数据。<br />通道关闭会通知所有正在读取通道的**协程**，相当于向所有读取协程中都写入了数据。<br />在实践中，并不需要关心是否所有的通道都已关闭，当通道没有被引用时将被 Go 语言的垃圾自动回收器**回收**。
> 关闭通道会触发所有通道读取操作被唤醒的特性，被使用在了很多重要的场景中，例如一个协程退出后，其创建的一系列子协程能够快速退出的场景。

关闭 channel 会调用到`[runtime.closechan()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L357)`函数：
```go
func closechan(c *hchan) {
    // 校验逻辑
    ...
    lock(&c.lock)
    // 设置 chan 已关闭
  c.closed = 1
  var glist gList
    // 获取所有接收者
  for {
    sg := c.recvq.dequeue()
    if sg == nil {
      break
    }
    if sg.elem != nil {
      typedmemclr(c.elemtype, sg.elem)
      sg.elem = nil
    }
    gp := sg.g
    gp.param = nil
    glist.push(gp)
  }
  // 获取所有发送者
  for {
    sg := c.sendq.dequeue()
    ...
  }
    unlock(&c.lock)
    // 唤醒所有glist中的goroutine
  for !glist.empty() {
    gp := glist.pop()
    gp.schedlink = 0
    goready(gp, 3)
  }
}
```
将`recvq`和`sendq`两个队列中的 Goroutine 加入到`gList`中，并清除所有`sudog`上未被处理的元素。最后将所有`glist`中的 Goroutine 加入调度队列，等待被唤醒。**注意，发送者在被唤醒之后会**`**panic**`。
<a name="lVqqf"></a>
# chan 作为参数和返回值
通道作为一等公民，可以作为参数和返回值。通道是协程之间交流的方式，不管是将数据读取还是写入通道，都需要将代表通道的变量通过函数传递到所在的协程中去。<br />通道作为返回值一般用于创建通道的阶段，下例中的`createWorker`函数创建了通道`c`，并新建了一个`worker`协程，最后返回的通道可能继续传递给其他的消费者使用。
```go
func worker(id int, c chan int){
    for n := range c {
        fmt.Printf("Worker %d received%c \n",id, n)
    }
}

func createWorker(id int) chan int {
    c := make(chan int)
    go worker(id, c)
    return c
}
```
由于通道是 Go 语言中的**引用类型**而不是值类型，因此传递到其他协程中的通道，实际引用了同一个通道。
<a name="uxxMW"></a>
# 小结
发送/接收/关闭操作可能引发的结果：

| **操作** | **nil channel** | **closed channel** | **channel** |
| --- | --- | --- | --- |
| **关闭 **`**close()**` | `panic` | `panic` | 成功关闭 |
| **发送 **`**ch <-**` | 阻塞 | `panic` | 阻塞 / 成功发送 |
| **接收 **`**<- ch**` | 阻塞 | 永远不阻塞 | 阻塞 / 成功接收 |

为什么会出现**阻塞**或者`panic`的情况呢？因为 Go 语言的 channel 以先进先出（`FIFO`）将资源分配给等待时间最长的 Goroutine，尽量消除数据竞争，让程序以尽可能顺序一致的方式运行。<br />关于让程序尽量顺序一致的含义，Go 语言内存模型采用传统的基于 **happens-before** 对读写竞争的定义：

1. 修改由多个 Goroutines 同时访问的数据的程序必须串行化这些访问。
2. 为了实现串行访问, 需要使用 channel 操作或其他同步原语（如`sync`/`atomic`包中的原语）来保护数据。
3. `go`语句创建一个 Goroutine，一定发生在 Goroutine 执行之前。
4. 往一个 channel 中发送数据，一定发生在从这个 channel 读取这个数据完成之前。
5. 一个 channel 的关闭，一定发生在从这个 channel 读取到零值数据（这里指因为`close`而返回的零值数据）之前。
6. 从一个无缓冲 channel 的读取数据，一定发生在往这个 channel 发送数据完成之前。

如果违反了这种定义，Go 会让程序直接 `panic` 或阻塞，无法往后执行。
<a name="iw4Xm"></a>
# Select

- 操作系统使用`select/poll/epoll`实现 IO 多路复用，通过起一个**线程**来监听并处理多个**文件描述符**代表的 TCP 连接，用来提高处理网络读写请求的效率。
- Go 语言使用`select`通过起一个`**goroutine**`监听多个`channel`（代表多个`goroutine`）的读写事件，用来提高从多个`channel`获取信息的效率。

**二者具体目标和实现不同，但本质思想都是相同的**。
<a name="gpghx"></a>
## 访问方式
对 Channel 的阻塞访问：

- 读：`i <- ch`或`i, ok <- ch`
- 写：`ch <- 1` 

对 Channel 的非阻塞访问：

- `select`关键字（解决因为一个通道阻塞而影响其他通道的正常读写问题）
> `select`语句所在的协程会陷入堵塞，直到一个或多个通道能够正常读写才恢复。

<a name="kgKlP"></a>
## 用法
<a name="Ve3tN"></a>
### 多 case + default
`select`中的多个`case`的表达式必须都是`channel`的读写操作，不能是其他的数据类型。

- 有多个`case`分支满足执行条件时，**随机选择**一个分支执行
```go
select {
    case <- chan1:
    // 如果 chan1 成功读到数据，则进行该 case 处理语句
    case chan2 <- 1:
    // 如果成功向 chan2 写入数据，则进行该 case 处理语句
    default:
    // 如果上面都没有成功，则进入default处理流程
}
```
除了`default`，还有一些其他的选择。例如，如果希望一段时间后仍然没有通道准备好则超时退出，可以选择`select`与**定时器**或者**超时器**配套使用。<br />如`<-time.After(800*time.Millisecond)`调用了`time`包的`After`函数，其返回一个通道 800ms 后会向当前通道发送消息，可以通过这种方式完成**超时控制**。
<a name="uj2LZ"></a>
### 多 case 无 default
程序会发生因所有`case`都不满足执行条件，且没有`default`分支而**阻塞**，Go 自带死锁检测机制，当发现所有协程都没有机会被唤醒时，则会发生 `panic`。
```go
select {
    case <- ch1:
    	// 从有缓冲chan中读取数据，由于缓冲区没有数据且没有发送者，该分支会阻塞
    	fmt.Println("Received from ch")
    case i := <- ch2:
    	// 从无缓冲chan中读取数据，由于没有发送者，该分支会阻塞
    	fmt.Printf("i is: %d", i)
}
```
<a name="KjZvc"></a>
### 单 case + default
`select`有一个`case`分支和`default`分支：

- 当`case`分支不满足执行条件时执行`default`分支
- 如果有满足执行条件的`case`分支，则执行对应的分支
```go
select {
    case <- ch1:
    	// 从有缓冲chan中读取数据，由于缓冲区没有数据且没有发送者，该分支会阻塞
    	fmt.Println("Received from ch")
    default:
    	fmt.Println("this is default")
}
```
<a name="uXrhB"></a>
### 无 case 永久阻塞
对于空的`select`语句，当前协程被**阻塞**，Go 自带死锁检测机制，当发现所有协程都没有机会被唤醒时，则会发生 `panic`。
```go
func main() {
  select {}
}


// 会发生程序因为 select 所在 goroutine 永久阻塞而失败的现象。

fatal error: all goroutines are asleep - deadlock!

goroutine 1 [select (no cases)]:
...
```
<a name="WaB9W"></a>
### for + select
`for`与`select`组合后，可以向`select`中加入一些**定时任务**。下例中的`tick`每隔 1s 就会向`tick`通道中写入数据，从而完成一些定时任务。
```go
c := make(chan int, 1)
tick := time.Tick(time.Second)
for {
    select {
        case <-c:
        	fmt.Println("Received from c")
        case <-tick:
            fmt.Println("Received from tick")
        case <-time.After(800 * time.Millisecond):
            fmt.Println("timeout")
    }
}
```
需要注意的是，定时器`time.Tick`与`time.After`是有本质不同的。

- `time.After`并不会定时发送数据到通道中，而只是在时间到了后发送一次数据。
> 当其放入`for`+`select`后，新一轮的`select`语句会重置`time.After`，这意味着第 2 次`select`语句依然需要等待800ms 才执行超时。如果在 800ms 之前，其他的通道就已经执行好了，那么`time.After`的case将永远得不到执行。

- `time.Tick`定时器在`for`循环的外部，因此其不重置，只会累积时间，实现定时执行任务的功能。
<a name="Ptay0"></a>
### select + nil
一个为`nil`的通道，不管是读取还是写入都将陷入阻塞状态。当`select`语句的`case`对`nil`通道进行操作时，`case`分支将**永远得不到执行**。<br />`nil`通道的这种特性，可以用于设计一些特别的模式。例如，假设有`a`、`b`两个通道，希望交替地向`a`、`b`通道中发送消息。
```go
func main() {
    a := make(chan int)
    b := make(chan int)

    go func(){
        for i := 0; i < 2; i++{
            select{
                case a<-i:
                	a = nil
                case b<-i:
                	b = nil
            }
        }
    }()
    fmt.Println(<-a)
    fmt.Println(<-b)
}
```
一旦写入通道后，就将该通道置为`nil`，导致再也没有机会执行该`case`，从而达到交替写入`a`、`b`通道的目的。
<a name="vtkjI"></a>
## 小结

1. `select`采用**多路复用**思想，本质是通过一个协程同时处理多个 IO 请求（`channel`读写事件）。
2. `select`基本用法是，通过多个`case`监听多个`channel`的读写操作：
   1. 任何一个`case`可以执行则选择该`case`执行
   2. 否则执行`default`
   3. 没有`default`，且所有的`case`均不能执行，则当前的 goroutine **阻塞**
3. **编译器**会对`select`有不同的`case`的情况进行优化以提高性能，尤其是避免对`channel`的加锁
   1. `select`没有`case`：直接调用运行时函数`**runtime.block()**`
   2. `select`有单`case`：直接转成对`channel`的操作`**runtime.chansend()/runtime.chanrecv()**`
   3. `select`有单`case`+`default`：以**非阻塞**方式对`channe`操作`**runtime.selectnbsend()/runtime.selectnbrecv()**`
4. 对最常出现的`select`有多`case`的情况
   1. 调用`**runtime.selectgo()**`函数来获取执行`case`的索引
   2. 生成`if`语句执行该`case`的代码
5. `selectgo()`函数的执行分为四个步骤：
   1. 生成轮询顺序（`pollorder`）和加锁顺序（`lockorder`）
      1. 随机生成一个遍历`case`的轮询顺序：避免`channel`饥饿，保证公平性
      2. 根据`channel`地址生成加锁顺序：避免死锁
   2. 根据轮询顺序查找`case`是否有可以立即收发的`channel`，有则获取`case`索引进行处理
   3. 如果轮询顺序上没有可以直接处理的`case`，则将当前`goroutine`加入各`case`的`channel`对应的收发队列上并等待其他`goroutine`的唤醒
   4. 当调度器唤醒当前`goroutine`时：
      1. 按加锁顺序遍历所有的`case`，查找需要被处理的`case`索引进行读写处理
      2. 从所有`case`的发送/接收等待队列中移除掉当前`goroutine`
<a name="f4YP0"></a>
## 底层原理
当`select`足够简单时，编译器将对其进行优化。例如，当`select`中只有一个控制通道的`case`语句时，和普通的通道操作是等价的。<br />`select`语句在运行时会调用核心函数`selectgo`，每个`case`在运行时都是一个`scase`结构体，存放了通道和通道中的元素类型等信息。
```go
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```
在`selectgo`函数中，有两个关键的序列，分别是`pollorder`和`lockorder`。

- `pollorder`代表乱序后的`scase`序列，这是一种类似洗牌算法的方式，将序列打散通过引入随机数的方式给序列带来了随机性。
- `lockorder`是按照大小对通道地址排序的算法，对所有的`scase`按照其通道在堆区的地址大小，使用了大根堆排序算法进行排序。

`selectgo`会按照该序列依次对`select`中所有的通道加锁。**而按照地址排序的目的是避免多个协程并发加锁时带来的死锁问题。**
```go
for i := range lockorder {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
```
<a name="DfdEd"></a>
### 一轮循环
当对所有`scase`中的通道加锁完毕后，`selectgo`开始了一轮对于所有`scase`的循环。循环的目的是找到当前准备好的通道。

- 如果`scase`的类型为`caseNil`，则会被忽略。
- 如果`scase`的类型为`caseRecv`，则和普通的通道接收一样，
   - 先判断是否有正在等待写入当前通道的协程，如果有则直接跳转到对应的`recv`分支，
   - 接着判断缓冲区是否有元素，如果有则直接跳转到`bufrecv`分支执行。
- 如果`scase`的类型为`caseSend`，则和普通的通道发送一样，
   - 先判断是否有正在等待读取当前通道的协程，如果有，则跳转到`send`分支执行，
   - 接着判断缓冲区是否有空余，如果有，则跳转到`bufsend`分支执行。
- 如果`scase`的类型为`caseDefault`，则会记录下来，并在循环完毕发现没有已经准备好的通道后，判断是否存在`caseDefault`类型，如果有，则跳转到`retc`分支执行。

一轮循环主要是找出是否有准备好的分支，如果有，则根据具体的情况执行不同的分支。这些分支和普通的通道很类似，主要是将元素写入或读取到当前的协程中，解锁所有的通道，并立即返回。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680860412557-2952d016-818d-42cd-8cce-6d087c6f2105.png#averageHue=%23eef7ea&clientId=u4b0d26b9-76ce-4&from=paste&height=499&id=u4de0d2fa&originHeight=998&originWidth=1370&originalType=binary&ratio=2&rotation=0&showTitle=false&size=471222&status=done&style=none&taskId=u91704578-ab27-4d02-a43f-acbb2da1d5c&title=&width=685)
<a name="RfwKz"></a>
### 二轮循环
当`select`完成一轮循环不能直接退出时，意味着当前的协程需要进入休眠状态并等待`select`中至少有一个通道被唤醒。不管是读取通道还是写入通道都需要创建一个新的`sudog`并将其放入指定通道的等待队列，之后当前协程将进入休眠状态。<br />当`select case`中的任意一个通道不再阻塞时，当前协程将被唤醒。要注意的是，最后需要将`sudog`结构体在其他通道的等待队列中出栈，因为当前协程已经能够正常运行，不再需要被其他通道唤醒。
<a name="XkkOw"></a>
# 参考
[从鹅厂实例出发！分析Go Channel底层原理](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651453784&idx=1&sn=9e00530e5be59b0ace1ef079884a05e6&chksm=80bb27aab7ccaebc8217a273fd7d289d977f10d302e19a9decdf2eb6e49674f6a7b73b87ac00&mpshare=1&scene=1&srcid=1202tJourbBENNxv7goRh6vc&sharer_sharetime=1669948929104&sharer_shareid=4213fd857d5e1093a89959d8b61544cb&version=4.0.19.70165&platform=mac#rd)<br />[最全Go select底层原理，一文学透高频用法](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651453947&idx=1&sn=30213b946751905535862e2a04715f35&chksm=80bb2709b7ccae1fa0062d21d17c7e82926278314b4fc2b683e892c9935fcb5c2a7708b443d7&mpshare=1&scene=1&srcid=0106EGKHFVip88joDz0avRXo&sharer_sharetime=1673005778184&sharer_shareid=4213fd857d5e1093a89959d8b61544cb&version=4.0.20.70171&platform=mac#rd)
