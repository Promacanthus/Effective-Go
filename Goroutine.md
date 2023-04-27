# Goroutine
Go 语言并发编程模型在底层是由**操作系统**所提供的**线程库**支撑的，这里先简要介绍一下线程实现模型的相关概念。
<a name="Obf6C"></a>
# 线程实现模型
线程的实现模型主要有3个，分别是：**用户级线程模型**、**内核级线程模型**和**两级线程模型**。**它们之间最大的差异在于用户线程与内核调度实体（Kernel Schedulable Entity, KSE）之间的对应关系上**。
> 内核调度实体就是可以被操作系统内核调度器调度的对象，也称为**内核级线程**，是操作系统内核的最小调度单元。

<a name="DjInx"></a>
## 用户级线程模型
![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669724907322-1fd5a938-d638-4b05-bcc0-9cd550eac5da.png#averageHue=%23f5f5f5&clientId=u4d1f06cc-ee01-4&from=paste&id=u3848dc14&originHeight=312&originWidth=652&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25251&status=done&style=none&taskId=ubd5e3f88-36df-4392-bfc6-68734e526b3&title=)<br />用户线程与 KSE 为多对一（`N:1`）的映射关系。<br />此模型下的线程由用户级别的线程库全权管理，线程库存储在进程的用户空间之中，这些线程的存在对于内核来说是无法感知的，所以这些线程也不是内核调度器调度的对象。<br />一个进程中所有创建的线程都只和同一个 KSE 在运行时动态绑定，**内核的所有调度都是基于用户进程的**。<br />对于线程的调度则是在用户层面完成的，相较于内核调度不需要让 CPU在 用户态和内核态之间切换，这种实现方式相比内核级线程模型可以做的很**轻量级**，对系统资源的消耗会小很多，上下文切换所花费的代价也会小得多。许多语言实现的**协程库**基本上都属于这种方式。<br />但是，**此模型下的多线程并不能真正的并发运行**。例如，如果某个线程在 `I/O` 操作过程中被阻塞，那么其所属进程内的所有线程都被阻塞，整个进程将被挂起。
<a name="nJONB"></a>
## 内核级线程模型
![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669724940361-935973db-33c1-451e-8374-f447cb574126.png#averageHue=%23f4f4f4&clientId=u4d1f06cc-ee01-4&from=paste&id=u3a5c9893&originHeight=311&originWidth=649&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26198&status=done&style=none&taskId=u00b70186-2cec-4b07-be32-3883ddfe3c3&title=)<br />用户线程与 KSE 为一对一（`1:1`) 的映射关系。<br />此模型下的线程由内核负责管理，应用程序对线程的创建、终止和同步都必须通过内核提供的系统调用来完成，内核可以分别对每一个线程进行调度。<br />所以，一对一线程模型可以真正的实现线程的并发运行，大部分语言实现的**线程库**基本上都属于这种方式。<br />但是，此模型下线程的创建、切换和同步都需要花费更多的内核资源和时间，如果一个进程包含了大量的线程，那么它会给内核的调度器造成非常大的负担，**甚至会影响到操作系统的整体性能**。
<a name="UrcRu"></a>
## 两级线程模型
![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669724956793-1e3d25cc-783d-4558-ac9e-be647e460d66.png#averageHue=%23f4f4f4&clientId=u4d1f06cc-ee01-4&from=paste&id=u11d9f677&originHeight=314&originWidth=648&originalType=binary&ratio=1&rotation=0&showTitle=false&size=29484&status=done&style=none&taskId=u2f326ec8-4a45-41f1-ae4d-64542a77d81&title=)<br />用户线程与 KSE 为多对多（`N:M`）的映射关系。<br />两级线程模型吸收前两种线程模型的优点并且尽量规避了它们的缺点：

1. 区别于用户级线程模型，两级线程模型中的进程可以与多个内核线程 KSE 关联，也就是说一个进程内的多个线程可以分别绑定一个自己的 KSE，这点和内核级线程模型相似；
2. 其次，又区别于内核级线程模型，它的进程里的线程并不与 KSE 唯一绑定，而是可以多个用户线程映射到同一个 KSE，当某个 KSE 因为其绑定的用户线程的阻塞操作被内核调度出 CPU 时，其关联的进程中其余用户线程可以重新与其他 KSE 绑定运行。

所以，两级线程模型既不是用户级线程模型那种完全靠自己调度的也不是内核级线程模型完全靠操作系统调度的，而是一种**自身调度与系统调度协同工作的中间态**，

- **用户调度器**实现用户线程到 KSE 的调度，
- **内核调度器**实现 KSE 到 CPU 上的调度。
<a name="eHzH5"></a>
# 设计目标
Goroutine 调度的设计目标：

1. 设计一种**高效**的并发编程模型：
   - 从开发的角度：只需要一个关键词（`go`）就能创建一个执行会话，即开发效率是高效的
   - 从运行态的角度：上述创建的会话能高效的被调度执行，即运行效率是高效的
2. 无大小限制的 Goroutine 栈
3. 公平的调度策略
<a name="VohqJ"></a>
# 设计历程
<a name="Gj3wj"></a>
## 多线程
实现并发的执行流，最直截了当的就是**多线程**，即每个 Goroutine 对应一个线程。

- 并发功能角度：可以实现并发
- 并发性能角度：性能会很差，尤其是在并发很重的时候，成千上万个线程的资源占用、创建销毁、调度带来的开销会很巨大
<a name="a0EwY"></a>
## 线程池
既然线程太多不好，那控制一下线程数量，使用**线程池**限定只启动 N 个线程。<br />那么就会出现 M 个 Goroutine，N 个线程，这种多对多的关系。<br />对于一个 Goroutine，它由哪个线程去执行，可以通过采用一个全局的 Global Runnable Queue，然后让所有线程主动去获取 Goroutine 来执行，那必然就会需要一个全局锁（mutex）来控制线程获取 Goroutine。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669690180078-3da8162b-7cf6-4dd9-ba59-2467e6c4fc16.png#averageHue=%23f9f9f9&clientId=u4d1f06cc-ee01-4&from=paste&height=322&id=uf3c06496&originHeight=644&originWidth=1418&originalType=binary&ratio=1&rotation=0&showTitle=false&size=99203&status=done&style=none&taskId=u4f7a2af7-e95b-4bc4-a19b-a14e15ef91c&title=&width=709)

- 控制了线程的数量，如果调度行为不是很频繁，可能问题不大
- 当线程较多时，就会有 **scalable** 的问题，**mutex** 的互斥竞争会非常激烈（考虑到基于时间片的抢占行为，实际上调度必然是很频繁的）
<a name="LwvpT"></a>
## 线程分治
在多线程编程领域中，**互斥**处理可以称得上是“名声在外”，需极其小心地去应对，最常见的解决方案：

1. 精妙地实现 lock free
2. 通过 “数据分治”和“逻辑分治”来避免做复杂的加锁互斥，将各个线程按横向（载荷分组）或纵向（逻辑划分）进行切分来处理工作。

在 Scheduler 的设计中，采用的是第二种方式，即数据分治的思想，也就是将 Goroutine 分而治之。每个线程分别处理一批 Goroutine，进行线程分治。将所有 Goroutine 分开放到各线程自己的存储中，即所谓的 Local Runnable Queue 中，同时 Global Runnable Queue 依然存在。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669690886427-171cb1e6-b65f-4787-8f39-075d3a334b8b.png#averageHue=%23f4f3f3&clientId=u4d1f06cc-ee01-4&from=paste&height=485&id=u4e3b6584&originHeight=970&originWidth=1376&originalType=binary&ratio=1&rotation=0&showTitle=false&size=173035&status=done&style=none&taskId=ubb0761a9-f972-4f5b-b9e7-2064c6c8430&title=&width=688)

- Local Runnable Queue 的引入，很大程度上解决了共享 **mutex** 带来的互斥竞争问题，但是并没有真正的解决 **scalable** 的问题。
- 为了充分利用 CPU，每个线程要按一定的策略去 Steal 其他线程 Local Runnable Queue 里面的 Goroutine 来执行，以避免线程之间出现负载不均的问题。那么在线程很多的时候，就会存在大量的无意义的 Steal 操作，因为此时其他线程的 Local Runnable Queue 可能也都是空的。
- 现在的内存资源是绑定在线程上面的，会导致线程数量和资源占用规模紧耦合。当线程数量多的时候，资源消耗也会比较大。
> 注：在 N 核的机器环境下，假如我们设定线程池大小为 N，由于系统调用的存在，实际的线程数量会超过 N。

<a name="RrYVK"></a>
## 资源池
既然每个线程一份资源也不合适，那么就仿照线程池的思路，单独做一个 Goroutine 资源池，做**计算存储分离**。把 Local Runnable Queue 及相关存储资源都挪出去，依然限定全局一共 N 份，即可**实现资源规模与系统中的真实线程数量的解耦**。<br />线程每次从对应的数据结构（Processor）中获取 Goroutine 去执行，Local Runnable Queue 及其他一些相关存储资源都挂在 Processor 下。<br />这样加一层 Processor 的抽象之后，便得到众所周知的 **GMP 模型**。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669692820387-d86aa267-3ca5-4516-97e4-dae5b2b1e319.png#averageHue=%23f7f7f7&clientId=u4d1f06cc-ee01-4&from=paste&height=377&id=ua20fc3b3&originHeight=754&originWidth=1380&originalType=binary&ratio=1&rotation=0&showTitle=false&size=581950&status=done&style=none&taskId=u4751b088-425d-4c2e-8d59-a942d1e5beb&title=&width=690)
<a name="pC6HG"></a>
## 公平与抢占
参考操作系统 CPU 的调度策略，通常各进程会分时间片，时间片用完了就轮到其他进程。在 Go 语言里也可以如此，不能让一些 Goroutine 长期霸占着运行资源不退出，必须实现基于时间片的“抢占”。<br />为了实现抢占就需要监测 Goroutine 执行时间片是否用完了。如果要检查系统中的各种状态变化、事件发生情况，通常会有**中断**与**轮询**两种思路：

- 中断：由一个中控方来做检查与控制
- 轮询：各个参与方按一定的策略主动 check 

因此对于 Goroutine 抢占而言，有以下两种解决方案：

- Signals：通过信号来中断原来的线程执行
- Cooperative checks：通过线程间歇性轮询自己，check 运行的时间片情况来主动暂停

Signals 和 Cooperative checks 对比：

| **Signal** | **Cooperative checks** |
| --- | --- |
| ✅ Fast | ❌ Slow（1-10%） |
| ❌ OS-dependent | ✅ OS-independent |
| ❌ non-preemptible regions | ✅ non-preemptible regions |
| ❌ GC stack/register maps | ✅ GC stack/register maps |

因为 Go 语言是自带 Runnabletime 的，而且代码编译生成也都是 go 编译器控制的，综合优劣分析，选择后者(Cooperative checks) 会比较合理。<br />对于 Cooperative checks 的方案，从**代码编译生成**的角度看，很容易做 check 指令的埋点。且因为 go 本来就要做动态增长栈，在函数入口处会插入检查是否该扩栈的指令，正好利用这一点来做相关的检查实现（这里有一些优化细节，可以使得基于时间片的抢占开销也较小）。
> 插入 check 指令的做法，会导致该方案存在一个理论缺陷：若有一个死循环，里面的所有代码都不包含check 指令，那依然会无法抢占，不过现实中基本不存在这种情况，总会做函数调用、访问 channel 等类似操作，因此不足为虑。

还有一个系统调用的问题，当线程一旦进入系统调用后，也会脱离 Runnabletime 的控制。万一系统调用阻塞了呢，基于 Cooperative checks 的方案，此时又无法进行抢占，是不是整个线程也就罢工了。<br />所以为了维持整个调度体系的高效运转，对于即将进入系统调用的线程，不做抢占，而是由它主动让出执行权。

1. 线程 A 在系统调用之前 handoff 让出 Processor 的执行权，唤醒一个 idle 线程 B 来做交接。
2. 当线程 A 从系统调用返回时，不会继续执行，而是将 G 放到 Runnable queue，然后进入 idle 状态等待唤醒，这样一来便能确保活跃线程数依然与 Processor 数量相同。
<a name="ujPFM"></a>
## 进一步的

- 在很多 cpu core 的情况下，活跃线程数比较多，work steal 的开销依旧有些浪费。
- 死循环不含 cooperative check 指令的这种 edge 情况的还没解决。
- 对于网络和 Timer 的 Goroutine 处理是使用全局方式的，不好 scale。
<a name="Gx550"></a>
# 设计思想
设计 Go 语言调度器的时候涉及到的一些软件设计思想如下：

- **线程池**，通过多线程提供更大的并发处理能力，同时又避免线程过多带来的过大开销。
- **资源池**，对有一定规模约束的资源进行池化管理，如内存池、机器池、协程池等，线程池也算作此类。
- **存算分离**，分别从逻辑、数据结构两个角度进行设计，规划二者的耦合关系。
- **中断与轮询**，用于监测系统中的各种状态变化、事件变化，通常来讲中断会比轮询更高效。
<a name="SKLOm"></a>
# 线程与协程
在 Go 语言中，协程被认为是轻量级的线程。和线程不同的是，操作系统内核感知不到协程的存在，协程的管理依赖 Go 语言运行时自身提供的调度器，同时，Go 语言中的协程是从属于某一个线程的。

- 调度方式：协程与线程的对应关系为`M:N`，即多对多。Go 语言调度器可以将多个协程调度到一个线程中，一个协程也可能切换到多个线程中执行。
- 上下文切换速度：协程的速度要快于线程，其原因在于协程切换不用经过操作系统用户态与内核态的切换，并且 Go 语言中的协程切换只需要保留极少的状态和寄存器变量值（SP/BP/PC），而线程切换会保留额外的寄存器变量值（例如浮点寄存器）。
> 上下文切换的速度受到诸多因素的影响，这里列出一些值得参考的量化指标：
> - **线程**切换的速度大约为1～2微秒
> - Go 语言中**协程**切换的速度比它快数倍，为0.2微秒左右

- 调度策略：
   - 线程的调度在大部分时间是抢占式的，操作系统调度器为了均衡每个线程的执行周期，会定时发出**中断信号**强制执行线程上下文切换。
   - Go 语言中的协程在一般情况下是协作式的，当一个协程处理完自己的任务后，可以主动将执行权限让渡给其他协程。
      - 这意味着协程可以更好地在规定时间内完成自己的工作，而不会轻易被抢占。
      - 当一个协程运行了过长时间时，Go 语言调度器才会强制抢占其执行。
- 栈大小：
   - 线程的栈大小一般是在创建时指定的，为了避免出现栈溢出（Stack Overflow），默认的栈会相对较大（例如`2MB`）。
> 这意味着每创建 1000 个线程就需要消耗`2GB`的虚拟内存，大大限制了线程创建的数量（64位的虚拟内存地址空间已经让这种限制变得不太严重）。

   - Go 语言中的协程栈默认为`2KB`，在实践中，经常会看到成千上万的协程存在。同时，线程的栈在运行时不能更改，但是协程栈在 Go 运行时的帮助下会动态检测栈的大小，并动态地进行扩容。因此，在实践中，可以**将协程看作轻量的资源**。
<a name="hawBY"></a>
# 并发与并行

- 并发（concurrency）：同时处理多个任务（独立的执行单元）的能力。并发并不意味着同一时刻所有任务都在执行，而是在一个**时间段**内，所有的任务都能执行完毕，开发者对任意时刻具体执行的是哪一个任务并不关心。
- 并行（parallelism）：同一时刻，多个任务都在执行。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1679994883058-0e600742-a143-496f-8bc9-e8beb78b9510.png#averageHue=%23faf8d2&clientId=u8fe1ecf9-0903-4&from=paste&height=613&id=u30f33082&originHeight=1226&originWidth=1436&originalType=binary&ratio=2&rotation=0&showTitle=false&size=314112&status=done&style=none&taskId=u95072398-b770-4f49-996f-bc0723a07e6&title=&width=718)

- 单核处理器中，
   - 任意一个**时刻，**只能执行一个具体的线程
   - 在一个**时间段**内，线程可能通过上下文切换交替执行
- 多核处理器是真正的**并行**执行，因为在任意时刻，可以同时有多个线程在执行。

在实际的多核处理场景中，并发与并行常常是同时存在的，即多核在并行地处理多个线程，而单核中的多个线程又在上下文切换中交替执行。<br />Go 语言中的协程依托于线程，

- 所以即便处理器运行的是同一个线程，在线程内 Go 语言调度器也会切换多个协程执行，这时协程是**并发**的。
- 如果多个协程被分配给了不同的线程，而这些线程同时被不同的 CPU 核心处理，那么这些协程就是**并行**的。

**因此在多核处理场景下，Go 语言的协程是并发与并行同时存在的。**但是，协程的并发是一种更加常见的现象，因为处理器的核心是有限的，而一个程序中的协程数量可以成千上万，这就需要依赖 Go 语言调度器合理公平地调度。
<a name="F5mYY"></a>
# GMP 模型
Go 语言中经典的 GMP 的概念模型生动地概括了线程与协程的关系：Go **进程**中的众多**协程**其实依托于**线程**，借助操作系统将线程调度到 CPU 执行，从而最终执行协程。
> 在 Go 的并发编程模型中，不受操作系统内核管理的独立控制流不叫用户线程或线程，而称为 Goroutine。Goroutine 通常被认为是协程的 Go 实现，实际上 Goroutine 并不是传统意义上的协程，传统的协程库属于用户级线程模型，而 Goroutine 结合 Go 调度器的底层实现上属于**两级线程模型**。

由 Go 调度器实现 Goroutine 到 KSE 的调度，由内核调度器实现 KSE 到 CPU 上的调度。Go 的调度器使用 G、M、P 三个结构体来实现 Goroutine 的调度，也称之为 **GMP 模型**。

- **G**：表示 Goroutine。每个 Goroutine 对应一个 G 结构体，G 存储 Goroutine 的运行堆栈、状态以及任务函数，**可重用**。
   - 当 Goroutine 被调离 CPU 时，调度器代码负责把 CPU 寄存器的值保存在 G 对象的成员变量之中，
   - 当 Goroutine 被调度起来运行时，调度器代码又负责把 G 对象的成员变量所保存的寄存器的值恢复到 CPU 的寄存器中。
- **M**：表示 OS 底层线程的抽象，它本身就与一个内核线程进行绑定，每个工作线程都有**唯一**的一个 M 结构体的实例对象与之对应，它代表着真正执行计算的资源，由操作系统的调度器调度和管理。M 结构体对象记录如下信息：
   1. 工作线程栈的起止位置
   2. 当前正在执行的 Goroutine 
   3. M 本身是否空闲等状态信息
   4. 通过指针维持与 P 结构体的实例对象之间的绑定关系
- **P**：表示逻辑处理器，方便协程调度与缓存。
   - 对 G 来说，P 相当于 CPU 核，G 只有绑定到 P（在 P 的 Local Runnable Queue 中）才能被调度。
   - 对 M 来说，P 提供了相关的执行环境（Context），如内存分配状态（mcache），任务队列（G）等。

它维护一个局部 Goroutine 可运行队列，工作线程优先使用自己的局部运行队列，只有必要时才会去访问全局运行队列，这可以大大减少锁冲突，提高工作线程的并发性，并且可以良好的运用程序的局部性原理。<br />GMP 模型本质是一个**存算分离**的架构设计，这里的存储是 Goroutine，计算是 Machine，Processor 就是用于关联和解耦存储和计算的中间层。<br />一个 G 的执行需要 P 和 M 的支持。一个 M 在与一个 P 关联之后，就形成了一个有效的 G 运行环境（内核线程+上下文）。每个 P 都包含一个可运行的 G 的队列（Local Runnable Queue）。该队列中的 G 会被依次传递给与本地 P 关联的 M，并获得运行时机。<br />在任一时刻，

- 一个 P 可能在其本地包含多个 G。
- 一个 P 只能绑定一个 M，但具体对应的是哪一个 M 是不固定的。
- 一个 G 并不是固定绑定同一个 P 的，有很多情况（例如 P 在运行时被销毁）会导致一个 P 中的 G 转移到其他的 P 中。
- 一个 M 可能在某些时候转移到其他的 P 中执行。

M 与 KSE 之间总是**一一对应**的关系，一个 M 仅能代表一个内核线程。M 与 KSE 之间的关联非常稳固，一个 M 在其生命周期内，会且仅会与一个 KSE 产生关联，而 M 与 P、P 与 G 之间的关联都是可变的。

- M 与 P 也是**一对一**的关系，
- P 与 G 则是**一对多**的关系。
<a name="EKjoP"></a>
## TIPS

- Local Runnable Queue 里面的 G 所创建的 G 会放到同样的 Local Runnable Queue（如果满了还是会放 GRQ），而且会限制被偷走，这样可以加强 Locality，同时为了保证公平也做了时间片继承，以免不停创建 G 会长期霸占运行资源。
- 被抢占的 G 会放到 Global Runnable Queue，GRQ 会每 61 次 tick 检查一次。
- G 的栈采用的是 Growable stack 方案，在函数入口会有栈检查的指令，如需扩容栈，会拷贝到新申请的更大的栈。
- Go Runtime 还会用 Background thread 来运行一些相对特别的 G（如 Network Poller、Timer）。
<a name="Lf92K"></a>
## G
G 在调度器中的地位与线程在操作系统中差不多，但是它占用了更小的内存空间，也降低了上下文切换的开销。它是 Go 语言在用户态提供的线程，作为一种粒度更细的资源调度单元，使用得当，能够在高并发的场景下更高效地利用机器的 CPU。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669797401893-4e6aa31d-5150-476c-ac07-a3778eeb4848.png#averageHue=%23f9f7f6&clientId=ufdd261bf-5f78-4&from=paste&id=u44d4885c&originHeight=821&originWidth=759&originalType=binary&ratio=1&rotation=0&showTitle=false&size=68037&status=done&style=none&taskId=u31638f33-469a-4670-a285-43d3992c7c5&title=)<br />`g` 结构体部分源码：
```go
type g struct {
    stack      		stack    	// Goroutine 的栈内存范围[stack.lo, stack.hi)
    stackguard0    	uintptr  	// 用于调度器抢占式调度
    m        		*m    		// Goroutine 占用的线程
    sched      		gobuf    	// Goroutine 的调度相关数据
    atomicstatus  	uint32  	// Goroutine 的状态
    ...
}

type gobuf struct {
    sp   	uintptr    		// 栈指针
    pc   	uintptr    		// 程序计数器
    g   	guintptr    	// gobuf 对应的 Goroutine
    ret   	sys.Uintewg  	// 系统调用的返回值
    ...
}
```
`gobuf` 中保存的内容会在调度器保存或恢复上下文时使用，其中栈指针和程序计数器会用来存储或恢复寄存器中的值，改变程序即将执行的代码。<br />`atomicstatus` 字段存储了当前 Goroutine 的状态，Goroutine 主要可能处于以下几种状态：

| **状态** | **状态** | **描述** | **是否在运行队列** | **是否在运行用户代码** | **是否拥有栈的所有权** | **备注** |
| --- | --- | --- | --- | --- | --- | --- |
| `_Gidle` | 刚开始创建时的状态 | just allocated and <br />has not yet been initialized |  |  |  | 当新创建的协程初始化后，会变为`_Gdead`状态 |
| `_Grunnable` | 在运行队列中，正在等待运行 |  | ✅ | ❌ | ❌ |  |
| `_Grunning` | 正在运行，已经分配了 P 和 M | assigned an M and a P <br />(g.m and g.m.p are valid) | ❌ | ✅ | ✅ |  |
| `_Gsyscall` | 正在执行系统调用 | executing a system call<br />assigned an M | ❌ | ❌ | ✅ |  |
| `_Gwaiting` | 在运行时被锁定，不能执行用户代码。<br />在垃圾回收及`channel`通信时经常会遇到这种情况。 | blocked in the runtime | ❌ | ❌ | ❌ | should be recorded somewhere (e.g., a channel wait queue) |
| `_Gdead` | 被销毁时的状态 | just exited, <br />on a free list, <br />or just being initialized | ❌ | ❌ | ❌ | It may or may not have a stack allocated |
| `_Gcopystack` | 在进行协程栈扫描时发现需要扩容或缩小协程栈空间，将协程中的栈转移到新栈时的状态 | stack is being moved | ❌ | ❌ | ✅ |  |
| `_Gpreempted` | 被强制抢占后的状态<br />Go 1.14新加的状态 | stopped itself for a suspendG preemption | ❌ | ❌ | ❌ | It is like `_Gwaiting`, but nothing is yet responsible for ready()ing it |
| `_Gscan` |  | GC is scanning the stack | ❌ | ❌ | ✅ | combined with one of the above states other than `_Grunning` |
| `_Gscanrunnable` |  |  |  |  |  |  |
| `_Gscanrunning` |  |  |  |  |  |  |

Goroutine 的状态迁移是一个十分复杂的过程，触发状态迁移的方法也很多。可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- **等待中**：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括`_Gwaiting`、`_Gsyscall`和`_Gpreempted`几个状态；
- **可运行**：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个Goroutine 就可能会等待更多的时间，即`_Grunnable`；
- **运行中**：Goroutine 正在某个线程上运行，即 `_Grunning`。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669800513940-c4bbbfe5-7de0-456a-a39a-d205182e33a9.png#averageHue=%23f8f4ef&clientId=uf1fe8af7-75a9-4&from=paste&id=ude739de8&originHeight=748&originWidth=1018&originalType=binary&ratio=1&rotation=0&showTitle=false&size=82799&status=done&style=none&taskId=ufb508e44-81a2-49ab-a5ef-3c7edf01540&title=)
<a name="vgLph"></a>
## M
M 是操作系统线程。调度器最多可以创建 10000 个线程，但是最多只会有 `GOMAXPROCS`（P 的数量）个**活跃线程**能够正常运行。在默认情况下，Runtime 会将`GOMAXPROCS`设置成当前机器的核数，也可以在程序中使用 `runtime.GOMAXPROCS` 来改变最大的活跃线程数。
> 例如，对于一个四核的机器，Runtime 会创建四个活跃的操作系统线程，每一个线程都对应一个`runtime.m` 结构体。在大多数情况下使用 Go 的默认设置，也就是线程数等于 CPU 数，默认的设置不会频繁触发操作系统的线程调度和上下文切换，所有的调度都会发生在用户态，由 Go 调度器触发，能够减少很多额外开销。

`m` 结构体部分源码：
```go

type m struct {
    g0      	*g      	// 一个特殊的 goroutine，执行 runtime 调度任务
    gsignal    	*g      	// 处理 signal 的 G
    curg    	*g      	// 当前 M 正在运行的 G 的指针
    p      		puintptr  	// 正在与当前 M 关联的 P
    nextp    	puintptr  	// 与当前 M 潜在关联的 P
    oldp    	puintptr  	// 执行系统调用之前使用线程的 P
    spinning	bool    	// 当前 M 是否正在寻找可运行的 G
    lockedg    	*g      	// 与当前 M 锁定的 G
}
```
`g0`表示一个特殊的 Goroutine，由 Go Runtime 在启动之初创建，它会深度参与 Runtime 调度过程，包括Goroutine 的创建、大内存分配和 CGO 函数的执行。
<a name="G0a4X"></a>
### 线程本地存储与线程绑定
线程本地存储是一种计算机编程方法，它使用线程本地的静态或全局内存。和普通的全局变量对程序中的所有线程可见不同，**线程本地存储中的变量只对当前线程可见**。因此，这种类型的变量可以看作是线程“私有”的。<br />一般地，操作系统使用`FS/GS`段寄存器存储线程本地变量。<br />在 Go 语言中，并没有直接暴露线程本地存储的编程方式，但是 Go 语言运行时的调度器使用线程本地存储将具体操作系统的线程与运行时代表线程的`m`结构体绑定在一起。
```go
type m struct{
	tls		[tlsSlots]uintptr // thread-local storage (for x86 extern register)
}
```
线程本地存储的实际是结构体`m`中`m.tls`的地址，同时`m.tls[0]`会存储当前线程正在运行的协程`g`的地址，因此在任意一个线程内部，通过线程本地存储，都可以在任意时刻获取绑定到当前线程上的协程`g`、结构体`m`、逻辑处理器`P`、特殊协程`g0`等信息。<br />通过线程本地存储可以实现结构体`m`与工作线程之间的绑定。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1680004265787-436bce9a-3a56-4bcf-9bfe-27449ea6bb15.png#averageHue=%23fbfbf5&clientId=ufd6dae69-9ff0-4&from=paste&height=328&id=ubd12f9c8&originHeight=656&originWidth=1394&originalType=binary&ratio=2&rotation=0&showTitle=false&size=196162&status=done&style=none&taskId=u05723c47-1a4b-4315-92ee-0c91ba66722&title=&width=697)
<a name="nqRbZ"></a>
## P
P 的数量等于 `GOMAXPROCS`，设置 `GOMAXPROCS` 的值只能限制 P 的最大数量，对 M 和 G 的数量没有任何约束。<br />调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 作时及时让出计算资源，提高线程的利用率。
> 当 M 上运行的 G 进入系统调用导致 M 被阻塞时，Runtime 会把该 M 和与之关联的 P 分离开来，这时，如果该 P 的 Local Runable Queue 上还有未被运行的 G，那么 Runtime 就会找一个空闲的 M，或者新建一个 M 与该 P 关联，满足这些 G 的运行需要。因此，M 的数量很多时候都会比 P 多。

`p` 结构体部分源码：
```go

type p struct {
  status   		uint32  		// p 的状态
  m        		muintptr    	// 对应关联的 M
  runqhead 		uint32  		// Local Runable Queue 可无锁访问
  runqtail 		uint32
  runq     		[256]guintptr
  runnext    	guintptr   		// 缓存可立即执行的 G
  gFree struct { 				// 可用的 G 列表，处于 _Gdead 状态的 G 
    gList
    n int32
  }
  ...
}

```
`P`可能处于的状态如下：

| **状态** | **描述** | **Owned By** | **Local Runable Queue** |
| --- | --- | --- | --- |
| `_Pidle` | not being used to run user code or the scheduler | idle list or whatever is transitioning its state | empty |
| `_Prunning` | used to run user code or the scheduler | M |  |
| `_Psyscall` | not running user code | It has affinity to an M in a syscall but is not owned by it and may be stolen by another M. |  |
| `_Pgcstop` | halted for STW | M |  |
| `_Pdead` | no longer used (GOMAXPROCS shrank)<br />reuse Ps if GOMAXPROCS increases |  |  |

![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669810776743-9c790aa7-68c0-4106-af53-9677dad8e64a.png#averageHue=%23faf8f4&clientId=u6c414008-caf2-4&from=paste&height=486&id=u6750484a&originHeight=972&originWidth=1512&originalType=binary&ratio=1&rotation=0&showTitle=false&size=98958&status=done&style=none&taskId=u1a614813-d574-42db-8e83-715a6b618a3&title=&width=756)
<a name="KhRrf"></a>
# 参考
[当谈论协程时，我们在谈论什么](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649774589&idx=1&sn=3860f40fed00c733bf43b4ce727d1f62&chksm=beccca8689bb4390fedd9083efc041c4b4a05aa821b1aa3848e62c81023819c861d05d0b014b&mpshare=1&scene=1&srcid=1126QVTbb9ywg4vfD7bDZ3ZY&sharer_sharetime=1669948976681&sharer_shareid=4213fd857d5e1093a89959d8b61544cb&version=4.0.19.70165&platform=mac#rd)<br />[Go scheduler: Implementing language with lightweight concurrency](https://2019.hydraconf.com/2019/talks/7336ginp0kke7n4yxxjvld/)
<a name="tti4o"></a>
# 
