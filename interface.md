# interface
<a name="d5ZBf"></a>
# 接口动态调用的效率
<a name="vKvh0"></a>
## 内存复制
虽然接口带来了开发效率的提升，但是很显然，接口没有直接调用函数速度快。<br />接口作为 Go 语言官方在语言设计时鼓励并推荐的习惯用法，在 Go 源代码中也经常看到它们的身影，这一事实已经足够让人相信接口动态调用的效率损失很小。实际上，在大部分情况下，接口的成本都可以忽略不计。<br />当方法的接收者是值或指针时，执行的效率是不同的。

- 当方法的接收者是值而不是指针时，因为接口中存储的值是逃逸到堆区的指针，因此，涉及从堆区复制值到栈中的过程，那么直接调用大约比接口动态调用快**两倍**。
- 当方法的接受者是指针时，直接调用与接口的动态调用间的**差距非常微小**。因此，执行接口的动态调用时，方法的接受者尽量使用指针。
<a name="uWNca"></a>
## 分支预测
影响接口动态调用效率的另一个因素是CPU 分支预测。CPU 能预取、缓存指令和数据，甚至可以预先执行，将指令重新排序、并行化等。

- 对于静态函数调用，CPU 会预知程序中即将到来的分支，并相应地预取必要的指令，这加速了程序的执行过程。
- 对于接口动态调用，CPU 无法预知程序的执行分支，为了解决此问题，CPU 应用了各种算法和启发式方法来猜测程序下一步将分支到何处（即**分支预测**），如果处理器猜对了，那么可以预期动态分支的效率几乎与静态分支一样，因为即将执行的指令已经被预取到了处理器的缓存中。

CPU 会根据 PC 寄存器里的地址，从内存里把需要执行的指令读取到指令寄存器中执行，然后根据指令长度自增，开始从内存中顺序读取下一条指令。而循环或者`if else`会根据判断条件产生两个分支，其中一个分支成立时对应着特殊的跳转指令，从而修改 PC 寄存器里面的地址，这样下一条要执行的指令就不是从内存里面顺序加载的，而另一个分支顺序读取内存中的指令。<br />由于动态调用的难以预测性，对于分支预测的担忧不是没有道理的。但这种在理论上存在的问题，在现实中极少成为问题。

1. 在现实中不会有如此密集分支预测错误导致性能下降的情况。
2. 现代 CPU 都有缓存，如果一个接口是经常使用的，那么其必然已经存在于 L1 缓存中，所以即便分支预测失败，仍然能够快速地获取接口的函数指针，而不必担心从主内存中获取数据额外花费的时间。
<a name="ll8J9"></a>
# 底层结构
<a name="qU5X8"></a>
## iface
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1679054151859-765c8f94-5843-44f0-8f6b-642fa4a5d037.png#averageHue=%23f0efcd&clientId=u3dca6d49-64c1-4&from=paste&height=339&id=uf69f70d4&originHeight=1356&originWidth=1492&originalType=binary&ratio=2&rotation=0&showTitle=false&size=406586&status=done&style=none&taskId=u19dacd8a-2083-48ba-acca-7898884ec78&title=&width=373)<br />带方法签名的接口在运行时的具体结构由`iface`构成，空接口的实现方式有所不同。
```go
type iface struct {
    tab *itab
    data unsafe.Pointer
}
```

- `tab`字段存储了接口的类型、接口中的动态数据类型、动态数据类型的函数指针等。
- `data`字段存储了接口中动态类型的函数指针。
> `data`字段存储了接口中具体值的指针，由于存储的数据可能很大也可能很小，这表明，存储在接口中的值必须能够获取其地址，所以平时分配在栈中的值一旦赋值给接口后，会发生内存逃逸，在堆区为其开辟内存。

<a name="a2AiF"></a>
## itab
```go
type itab struct {
    inter *interfacetype
    _type *_type
    hash uint32
    _ [4]byte
    fun [1]uintptr
}
```

- `inter`字段代表接口本身的类型，类型`interfacetype`是对`_type`的简单包装。
- `_type`字段代表接口存储的动态类型。
<a name="tn9N7"></a>
## iterfacetype
```go
type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}
```
除了类型标识`_type`，还包含了一些接口的元数据。

- `pkgpath`代表接口所在的包名，
- `mhdr []imethod`表示接口中暴露的方法在最终可执行文件中的名字和类型的偏移量。通过此偏移量在运行时能通过`resolveNameOff`和`resolveTypeOff`函数快速找到方法的名字和类型。
<a name="JduM8"></a>
## _type
Go语言的各种数据类型都是在`_type`字段的基础上通过增加额外字段来管理的，如切片与结构体。
```go
type slicetype struct {
    typ _type
    elem *_type
}

type structtype struct {
    typ _type
    pkgPath name
    field []structfield
}
```
`_type`包含了类型的大小、哈希、标志、偏移量等元数据。
```go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```
<a name="QnyHg"></a>
# 示例
```go
type Shape interface {
    perimeter() float64
    area() float64
}

type Rectangle struct {
    a, b float64
}

func (r Rectangle) perimeter() float64 {
    return (r.a + r.b) * 2
}

func (r Rectangle) area() float64 {
    return r.a * r.b
}

func main() {
    var s Shape
    s = Rectangle(3, 4)
}
```
接口变量`s`在内存中如下图所示。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1679054591107-9f305b28-5628-45e9-88cf-d354d89d7fc3.png#averageHue=%23f9f8f2&clientId=u3dca6d49-64c1-4&from=paste&height=161&id=ua7e10773&originHeight=322&originWidth=1456&originalType=binary&ratio=2&rotation=0&showTitle=false&size=139647&status=done&style=none&taskId=uf1478bee-9327-4e77-ac56-e664378ad35&title=&width=728)<br />从图中可以看出，接口变量中存储了接口本身的类型元数据，动态数据类型的元数据、动态数据类型的值及实现了接口的函数指针。
<a name="PnYTo"></a>
# 空接口
空接口中没有任何方法签名，可以容纳任意的数据类型，在实际中经常被使用（例如格式化输出），因此需要考虑**任何类型转换为空接口的性能**。
> 空接口的笨重主要由于：
> 1. 内存逃逸的消耗
> 2. 创建`eface`的消耗
> 3. 为堆区的内存设置垃圾回收
> 
如果赋值的对象一开始就已经分配在了堆中，则不会有如此夸张的差别。

和有方法的接口（`iface`）相比，空接口不需要`interfacetype`表示接口的内在类型，也不需要`func`方法列表。<br />对于空接口，Go 语言在运行时使用了特殊的`eface`类型，其在 64 位系统中占据 16 字节。
```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```
<a name="PXZrp"></a>
# 陷阱
当要使用`nil`判断接口是否为空时，不要使用如下的中间层对接口进行包装。
```go
func do() *os.PathError {
    return nil
}

func warpDo() error {
    return do()
}

warpDo() == nil // false
```
接口为`nil`与接口值为`nil`是有区别的，由于接口`error`具有动态类型`*os.PathError`，因此`err==nil`为`false`。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1679057824889-ee8167ae-2e63-4a91-8cd8-1dabf65b5d4f.png#averageHue=%23dff1d0&clientId=ub55a19a1-e483-4&from=paste&height=353&id=u252876a5&originHeight=1410&originWidth=1458&originalType=binary&ratio=2&rotation=0&showTitle=false&size=571951&status=done&style=none&taskId=u003823f5-6484-406e-aefb-1d76c937781&title=&width=365)
