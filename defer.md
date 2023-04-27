# defer
`defer`是 Go 语言中的关键字，也是 Go 语言的重要特性之一。<br />`defer`的语法形式为`defer Expression`，其后必须紧跟一个函数调用或者方法调用，不能用括号括起来。在很多时候，`defer`后的函数以匿名函数或闭包的形式呈现，
```go
defer func(...){
    // Done
}()
```
`defer`将其后的函数推迟到了其所在**函数返回之前**执行。不管`defer`函数后的执行路径如何，最终都将被执行，一般用于：

- 资源释放
- 异常（`panic`）捕获

Go 语言中的`defer`有一些自己的特点及灵活性：

1. 程序中可以有多个`defer`
2. `defer`的调用可以存在于函数的任何位置
3. `defer`可能不会被执行，例如，如果判断条件不成立则放置在`if`语句中的`defer`可能不会被执行。

`defer`语句在使用时有许多陷阱，Go语言中`defer`的实现方式也经历了多次演进。
<a name="g2RU2"></a>
# defer使用场景
`defer`给 Go 代码的书写方式带来了很大的变化。
<a name="v4F1O"></a>
## 资源释放
在程序中使用`os.Open()`及`os.Create()`打开文件资源描述符，并在最后通过`file.Close()`方法得到释放。

- 在正常情况下，该程序能正常运行，
- 一旦在文件创建过程中出现错误，程序就直接返回，文件描述符将得不到释放。

因此需要在所有错误退出时释放资源才能保证其在异常情况下的正确性。同样的问题也存在于锁的释放中，一旦在加锁后发生了错误，就需要在每个错误的地方都释放锁，否则会出现严重的死锁问题。<br />Go语言中的`defer`特性能够很好地解决这一类资源释放问题，不管`defer`后面的执行路径如何，`defer`中的语句都将执行。<br />在每个资源打开后都立即加入`defer file.Close()`函数，保证函数在任意路径执行结束后都能够关闭资源。<br />`**defer**`**是一种优雅的关闭资源的方式，能减少大量冗余的代码并避免由于忘记释放资源而产生的错误。**
<a name="DpyVE"></a>
## 异常捕获
程序在运行时可能在任意的地方发生`panic`异常，例如算术除 0 错误、内存无效访问、数组越界等，这些错误会导致程序异常退出。<br />很多时候，我们希望能够捕获这样的错误，同时希望程序能够继续正常执行。`defer`的特性是无论后续函数的执行路径如何以及是否发生了`panic`，在函数结束后一定会得到执行，这为异常捕获提供了很好的时机。异常捕获通常结合`recover()`函数一起使用。
```go
package main

import "fmt"

func main() {
    executePanic()
    fmt.Println("Main block is executed completely.")
}

func executePanic() {
    defer fmt.Println("defer func.")
    panic("This is panic situation.")
    fmt.Println("The function is executed completely.")
}
```
上面的实例代码在`executePanic()`函数中，手动执行`panic`函数触发了异常。<br />当异常触发后，函数仍然会调用`defer`中的函数，然后异常退出。<br />输出如下，表明调用了`defer`中的函数，并且`main`函数将不能正常运行，程序异常退出打印出栈追踪信息。
```go
defer func.
panic: This is panic situation.

goroutine 1 [running]:
main.executePanic()
	/Users/zhubowen/Documents/github.com/kubernetes/tempCodeRunnerFile.go:12 +0x73
main.main()
	/Users/zhubowen/Documents/github.com/kubernetes/tempCodeRunnerFile.go:6 +0x19
exit status 2

```
当在`defer`中使用`recover()`函数进行异常捕获后，程序将不会异常退出，并且能够执行正常的函数流程。
```go
package main

import "fmt"

func main() {
    executePanic()
    fmt.Println("Main block is executed completely.")
}

func executePanic() {
    defer func ()  {
        if err := recover();err!=nil{
            fmt.Println(err)
        }
        fmt.Println("This is recovery function.")
    }()
    panic("This is panic situation.")
    fmt.Println("The function is executed completely.")
}
```
如下输出表明，尽管有`panic`，`main`函数仍然在正常执行后退出。
```go
This is panic situation.
This is recovery function.
Main block is executed completely.
```
<a name="BmFRe"></a>
# defer 特性
<a name="XT2FI"></a>
## 延迟执行
`defer`后的函数并不会立即执行，而是推迟到了函数结束后执行。<br />延迟执行的特性可用于函数的中间件。例如，对 HTTP 服务器进行简单的中间件封装。
```go
func recoverHandler(next http.Handler) http.Handler {
	fn := func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("panic: %+v", err)
				http.Error(w, http.StatusText(500), 500)
			}
		}()
		next.ServerHTTP(w, r)
	}
	return http.HandlerFunc(fn)
}
```
如上，该中间件的目的是捕获所有的`http`操作执行异常，并向客户端返回 500 异常。在编码是可以轻松地加入`recoverHandler`中间件而不破坏代码原本的结构。<br />还可以设计一种类似计算函数执行时间的日志，同样依靠`defer`函数实现。
<a name="uYGjk"></a>
## 参数预计算
`defer`的**参数预计算**特性常导致使用`defer`时犯错。因为大部分时候，只记住了**延迟执行**特性。<br />参数的预计算指当函数到达`defer`语句时，延迟调用的参数将立即求值，传递到`defer`函数中的参数将预先被固定，而不会等到函数执行完成后再传递参数到`defer`中。
```go
package main

import "fmt"

func main(){
	a := 1
	defer func(b int){
		fmt.Println("defer b:",b)
	}(a+1)
	a=99
}

// defer b: 2
```
如上代码，`defer`后的函数需要传递`int`参数：

1. 首先`a := 1`
2. 接着`defer`函数的参数传递为`a+1`
3. 最后，在函数返回前`a = 99`
4. 最后`defer`函数打印出的`b`值是`2`

原因是传递到`defer`的参数是预j计算的，因此在执行到`defer`语句时，执行了`a+1`并将其保留了起来，直到函数执行完成后才执行`defer`函数体内的语句。
<a name="IDw2o"></a>
## 执行顺序(LIFO)
当函数体内部出现多个`defer`函数时，按照后入先出（last-in first-out，LIFO）的顺序执行，这与栈的执行顺序是相同的。<br />这也意味着后申请的资源将会先得到释放，如下：

- 加锁的顺序为：`m→n`
- 释放锁的顺序为：`n→m`

这与习惯中资源释放的方式是相同的。
```go
m.Lock()
defer m.Unlock()
n.Lock()
defer n.Unlock()
```
<a name="Ot7xE"></a>
# defer返回值与陷阱
除了参数预计算的坑，`defer`还有一种非常容易犯错的场景，涉及与返回值参数结合。<br />如下所示，函数`f`中有返回值`r`，`return g`之后在`defer`函数中将`g`赋值为`200`。
```go
package main

import "fmt"

var g = 100

func f() (r int) {
    defer func() {
        g = 200
    }()

    fmt.Printf("f: g = %d\n", g)
    return g
}

func main() {
    i := f()
    fmt.Printf("main: i = %d, g = %d\n", i, g)
}

// f: g = 100
// main: i = 100, g = 200
```
从输出结果可以推测出，在`return`之后，执行了`defer`函数。
```go
package main

import "fmt"

var g = 100

func f() (r int) {
    r = g
	defer func() {
		r = 200
	}()

    r = 0
	fmt.Printf("f: r = %d\n", r)
	return r
}

func main() {
	i := f()
	fmt.Printf("main: i = %d, g = %d\n", i, g)
}


// f: r = 0
// main: i = 200, g = 100
```
对上例中的代码稍做修改，从输出结果可以推测出，在`defer`执行完成后，执行了`return`，因为其返回值`r`是在`defer`函数中赋值的。<br />上述两种截然相反的结果，是因为`return`不是一个原子操作，其包含了下面几步：

1. 将返回值保存在栈上
2. 执行`defer`
3. 执行`return`
```go
// 第1个例子中的函数f可以翻译为如下伪代码，最终返回值r为100。
g = 100
r = g
defer g = 200
return i = r = 100

// 第2个例子中的函数f可以翻译为如下伪代码，最终返回值r为200。
g = 100 
r = g
r = 0
defer r = 200
return i = r = 200
```
<a name="nf3A9"></a>
# defer演进
Go 语言中`defer`的实现经历了复杂的演进过程，`Go 1.13`、`Go 1.14`都经历了比较大的更新。<br />在`Go 1.13`之前，`defer`是被分配在**堆区**的，尽管有全局的缓存池分配，仍然有比较大的性能问题，原因在于：

1. 使用`defer`涉及堆内存的分配
2. 需要存储`defer`函数中的参数
3. 需要将堆区数据转移到栈中执行，涉及内存复制

因此，`defer`比普通函数的直接调用要慢很多。<br />为了将调用`defer`函数的成本降到与调用普通函数相同，在`Go 1.13`中：

1. 在大部分情况下，将`defer`语句放置在了**栈**中，避免在堆区分配、复制对象。
2. 但仍然和`Go 1.12`一样，将整个`defer`语句放置到一条链表中，从而能够在函数退出时，以**LIFO**的顺序执行。
>  将`defer`添加到链表中是必不可少的，因为：
> 1. `defer`的数量可能是无限的
> 2. 也可能是动态调用，如通过`for`或者`if`包裹的`defer`语句，只有在运行时才能决定执行的个数。

在`Go 1.13`中包括两种策略：

1. 对于最多调用一个（at most once）语义的`defer`语句使用了**栈分配**策略，
2. 对于其他的方式，例如，`for`循环体内部的`defer`语句，采用**堆分配**策略。

在大部分情况下，程序中的`defer`涉及的都是比较简单的场景，这一改变也大幅度提高了`defer`的效率。`defer`的操作时间从`Go 1.12`时的`50ns`降到`Go 1.13`时`35ns`（直接调用大约花费`6ns`）。虽然进行了一定程度的优化，但仍然比直接调用慢了5、6倍左右。<br />在`Go 1.14`进一步对最多调用一个的`defer`语义进行了优化，实现编译时内联。因此，根据不同的场景，实际存在了三种实现`defer`的方式。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1677760098888-3811b2a8-f4ef-4e92-8575-82cfd472c66b.png#averageHue=%23f3f3f3&clientId=u4d2aa8b5-66d5-4&from=paste&height=179&id=ua32247d0&originHeight=358&originWidth=1424&originalType=binary&ratio=2&rotation=0&showTitle=false&size=111514&status=done&style=none&taskId=u7b236c4c-9e71-4855-9f99-6d9a3989d32&title=&width=712)<br />如果使用了`Go 1.14`以上版本，则可以认为`defer`与直接调用的效率相当，不用为了考虑高性能而纠结是否使用`defer`。
