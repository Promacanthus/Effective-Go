# panic
`defer`正常执行的流程是：

1. 执行函数体
2. 执行`defer`语句
3. 执行`return`语句

但程序可能异常退出，引起异常退出的原因可能是：

1. 用户代码中错误的状态
2. 运行时数组越界、哈希表读写冲突
3. 访问无效内存

有时候，不希望程序异常退出，而是希望捕获异常并让函数正常执行，这涉及`defer`、`recover`的结合使用。
<a name="MQhgz"></a>
# panic()
在Go语言中，有以下两个内置函数可以处理程序的异常情况：
```go
func panic(interface{})
func recover(interface{})
```
`panic`函数传递的参数为空接口`interface{}`，其可以存储任何形式的错误信息并进行传递。在异常退出时会打印出来。<br />Go程序在`panic`时，不会导致程序异常退出，而是会终止当前函数的正常执行，**执行**`**defer**`**函数并逐级返回。**
> 除了手动触发`panic`，在Go语言运行时的一些阶段也会检查并触发`panic`，例如：
> 1. 数组越界（`runtime error：index out of range`）
> 2. `map`并发冲突（`fatal error：concurrent map read and map write`）

<a name="Q5rTA"></a>
# recover()
虽然实际项目中都有让程序崩溃之后重新启动的机制（例如 kubernetes pod 重启），但这仍然需要时间，在这期间可能丢失重要的数据和用户体验，所以很多时候不希望程序异常退出。<br />为了让程序在`panic`时仍然能够执行后续的流程，Go语言提供了内置的`recover`函数用于异常恢复。

1. `recover()`函数一般与`defer`函数结合使用才有意义，其返回值是`panic`中传递的参数。
2. `panic`会调用`defer`函数，因此，在`defer`函数中可以加入`recover()`起到让函数恢复正常执行的作用。
<a name="I0Ac3"></a>
# panic与recover嵌套
`panic`会遍历`defer`链并调用，那么如果在`defer`函数中发生了`panic`会怎么样呢？如下所示：

1. 函数`a`触发了`panic("a panic")`后调用`defer`函数`b`
2. 函数`b`触发了`panic("b panic")`调用`defer`函数`fb`
3. 函数`fb`同样触发`panic("fb panic")`
```go
package main

func main() {
	a()
}

func a()  {
	defer b()
	panic("a panic")
}

func b()  {
	defer fb()
	panic("b panic")
}

func fb()  {
	panic("fb panic")
}
```
最终程序输出如下，先打印最早出现的`panic`，再打印其他的`panic`。其实，每一次`panic`调用都新建了一个`_panic`结构体，并用一个链表存储。
```go
panic: a panic
	panic: b panic
	panic: fb panic

goroutine 1 [running]:
main.fb()
	/xxx/main.go:18 +0x27
panic({0x105bb00, 0x107a968})
	/usr/local/go/src/runtime/panic.go:884 +0x213
main.b()
	/xxx/main.go:14 +0x49
panic({0x105bb00, 0x107a958})
	/usr/local/go/src/runtime/panic.go:884 +0x213
main.a()
	/xxx/main.go:9 +0x49
main.main()
	/xxx/main.go:4 +0x17
exit status 2
```
**嵌套**`**panic**`**不会陷入死循环，每个**`**defer**`**函数都只会被调用一次。**<br />当嵌套的`panic`遇到了`recover`时，情况变得更加复杂。将上面的程序稍微改进一下，让`main`函数捕获嵌套的`panic`。
```go
package main

import "fmt"

func main() {
	defer catch("main")
	a()
}

...

func catch(funcName string)  {
	if r := recover();r!=nil{
		fmt.Println(funcName,"recover:",r)
	}	
}

// main recover: fb panic
```
最终程序的输出结果为`main recover：fb panic`，这意味着`recover()`函数最终捕获的是最近发生的`panic`，即便有多个`panic`函数，在最上层的函数也只需要一个`recover()`函数就能让函数按照正常的流程执行。<br />如果`panic`发生在函数`b`或函数`fb`中，则情况会有所不同。<br />例如，将函数`fb`改写如下，内部的`recover`只能捕获由当前函数或其子函数触发的`panic`，而不能触发上层的`panic`。
```go
...

func fb()  {
	defer catch("fb")
	panic("fb panic")
}
...

// fb recover: fb panic
// main recover: c panic
```
<a name="yU6XK"></a>
# panic() 底层原理
`panic`函数在编译时会被解析为`runtime.gopanic()`函数，每调用一次`panic`都会创建一个`_panic`结构体。
```go
// A _panic holds information about an active panic.
//
// A _panic value must only ever live on the stack.
//
// The argp and link fields are stack pointers, but don't need special
// handling during stack growth: because they are pointer-typed and
// _panic values only live on the stack, regular stack pointer
// adjustment takes care of them.
type _panic struct {
	argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
	arg       any            // argument to panic
	link      *_panic        // link to earlier panic
	pc        uintptr        // where to return to in runtime if this panic is bypassed
	sp        unsafe.Pointer // where to return to in runtime if this panic is bypassed
	recovered bool           // whether this panic is over
	aborted   bool           // the panic was aborted
	goexit    bool
}
```
与`_defer`结构体一样，`_panic`也会被放置到当前协程的链表中，原因是`panic`可能发生嵌套，例如`panic → defer → panic`，因此可能同时存在多个`_panic`结构体。
```go
func gopanic(e any) {
	gp := getg()
    ...
    var p _panic
    p.arg = e
    p.link = gp._panic
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))
	runningPanicDefers.Add(1)
    ...
}
```
在正常情况下`panic`的执行流程：

1. 遍历协程中的`defer`链表
2. 对于通过**堆**或**栈**分配实现的`defer`语句，通过反射的方式调用`defer`中的函数
> `panic`通过`reflectcall()`调用`defered()`函数而不是直接调用的原因：
> - 直接调用`defered()`函数需要在当前栈帧中为它准备参数
> - 不同`defered()`函数的参数大小可能有很大差异
> - `gopanic`函数的栈帧大小固定而且很**小**，可能没有足够的空间来存放`defered()`函数的参数

在正常情况下，当`defer`链表遍历完毕后，`panic`会退出。
<a name="FBvV6"></a>
# recover() 底层原理
不管`defer`采用了哪一种策略，在正常情况下，`panic`都会遍历`defer`链并退出。当在`defer`中使用了`recover`异常捕获之后，情况又变得复杂许多。<br />内置的`recover`函数在运行时将被转换为调用`runtime.gorecover()`函数。在正常情况下将当前对应的`_panic`结构体的`recovered`字段设置为`true`并返回。
```go
func gorecover(argp uintptr) any {
	gp := getg()
	p := gp._panic
	if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```

- `gorecover()`函数的参数`argp`为其调用者函数的参数地址
- `p.argp`为发生`panic`时`defer`函数的参数地址
- `argp == uintptr(p.argp)`可用于判断`panic`和`recover`是否匹配，因为内层`recover`不能捕获外层的`panic`

`gorecover()`并没有进行任何异常处理，真正的处理发生在`runtime.gopanic()`函数中。<br />在遍历`defer`链表执行的过程中：

1. 一旦发现`p.recovered`为`true`，就代表当前`defer`中调用了`recover`函数
2. 删除当前链表中为内联`defer`的`_defer`结构
3. 之后程序恢复正常的流程，（内联`defer`直接通过内联的方式执行）

`gopanic()`函数最后会调用`mcall(recovery)`，`mcall`将切换到`g0`栈执行`recovery()`函数。
> `recovery()`函数接受从`defer`中传递进来的`SP`、`PC`寄存器并借助`gogo()`函数切换到当前协程继续执行。
> `gogo()`函数是与操作系统有关的函数，用于完成栈的切换及`CPU`寄存器的恢复。

当前协程的`SP`、`PC`寄存器被修改为了从`defer`中传递进来的`SP`、`PC`寄存器，从而改变了函数的执行路径。<br />总之，`recover()`通过修改协程`SP`、`PC`寄存器值使函数重新执行`deferreturn()`函数。`deferreturn()`函数的作用是继续执行剩余的`defer()`函数（因为有一部分`defer`函数可能已经在`gopanic()`函数中得到了执行），并返回到调用者函数，就像程序并没有`panic`一样。
