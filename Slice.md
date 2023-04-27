# Slice
Go 语言中的切片（`slice`）在某种程度上和其他语言（例如 C 语言）中的数组在使用中有许多相似的地方。但是 Go 语言中的切片有许多独特之处，例如，切片是长度可变的序列。序列中的每个元素都有相同的类型。一个切片一般写作`[]T`，其中`T`代表`slice`中元素的类型。和数组不同的是，切片不用指定固定长度。
<a name="SXG0B"></a>
# 切片
<a name="kuGse"></a>
## 切片结构
切片是一种轻量级的数据结构，提供了访问数组**任意**元素的功能，在 Go 语言中切片不是数组，而是一个结构体，其定义如下：
```go
// SliceHeader is the runtime representation of a slice.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
//
// In new code, use unsafe.Slice or unsafe.SliceData instead.
type SliceHeader struct {
	Data uintptr	// 指向存放数据的数组指针
	Len  int		// 长度有多大
	Cap  int		// 容量有多大
}
```
一个空的切片的表现如下图所示，在运行时由指针（`date`）、长度（`len`）和容量（`cap`）3 部分构成。

- 指针：指向切片元素对应的底层数组元素的地址。
- 长度：对应切片中元素的数目，长度不能超过容量。
- 容量：一般是从切片的开始位置到底层数据的结尾位置的长度。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1673277926667-d4274878-a9fb-4276-b55a-de4f4edda674.png#averageHue=%23f6f6f0&clientId=u77beecb5-d556-4&from=paste&height=222&id=uad7e1715&originHeight=295&originWidth=466&originalType=url&ratio=1&rotation=0&showTitle=false&size=17458&status=done&style=none&taskId=u5a14b786-fc9e-47b9-ac56-2f75aa9cbbf&title=&width=350)
> 在结构体里用**数组指针**的问题—**数据会发生共享。**

<a name="hVKly"></a>
## 切片初始化
切片有多种声明方式，初始化需要使用内置的`make()`函数。切片有长度和容量的区别，可以在初始化时指定，内置的`len()`和`cap()`函数可以分别获取切片的长度和容量。<br />由于切片具有可扩展性，所以当它的容量比长度大时，意味着为切片元素的增长预留了内存空间。
```go
var slice1 []int
var slice2 = make([]int,5)
var slice3 = make([]int,3,5)
slice4 := []int{1,2,3}
```

- 在只声明不赋初始值的情况下，切片`slice1`的值为`nil`。
- `slice2`指定了长度为 5 的`int`切片，如果不指定容量，则默认其容量与长度相同。
- `slice4`称为切片字面量，在初始化阶段对切片进行了赋值。编译器会自动推断出切片初始化的长度，并使其容量与长度相同。
<a name="vbosP"></a>
## 切片截取
和数组一样，切片中的数据也是内存中的一片连续区域。要获取切片某一区域的连续数据，可以通过下标的方式对切片进行截断，被截取后的切片，其长度和容量都发生了变化。<br />切片在被截取时的另一个特点是，**被截取后的数组仍然指向原始切片的底层数据**。
```go
foo := make([]int,5)
foo[3] = 43
foo[4] = 100
bar := foo[1:4]
bar[1] = 99
```
`bar`截取了`foo`切片中间的元素，并修改了`bar`中的第 2 号元素，程序执行完成后，其底层结构如下所示。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1676619800768-a932774b-1dca-4c5b-9fc1-4fdc9a13f0ee.png#averageHue=%23f3f2d9&clientId=u86338f73-83d8-4&from=paste&height=303&id=u06273dfe&originHeight=606&originWidth=1442&originalType=binary&ratio=2&rotation=0&showTitle=false&size=283775&status=done&style=none&taskId=u3a63ac1a-92a7-43d9-8ede-c1824b7ffaa&title=&width=721)
<a name="Eor5r"></a>
## 切片复制与数值引用
数组的复制是**值复制**，对于数组副本的修改不会影响到原数组。
```go
a := [...]int{1,2,3,4}
b := a
b[1] = 100
fmt.Println("a=",a,"b=",b) // a=[1,2,3,4] b=[1,200,3,4]
```
但是,对于切片的副本的修改会影响到原来的切片，这说明切片的副本与原始切片共用一个内存空间。
```go
c := []int{100,200,300}
d := c
d[0] = 1
fmt.Println("c=",c,"d=",d) // c=[1,200,300] d=[1,200,300]
```
Go 语言中，切片的复制也是**值复制**，但这里的**值复制**是对于运行时`SliceHeader`结构的复制。<br />如下图所示，底层指针仍然指向相同的底层数据的数组地址，可以理解为数据进行了**引用传递**。切片的这一特性使得即便切片中有大量数据，在复制时的成本也比较小，这与数组有显著的不同。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1676620587633-003697cd-f031-4be8-aa3d-79247f912bea.png#averageHue=%23f7f7f7&clientId=u86338f73-83d8-4&from=paste&height=280&id=u7d1836b0&originHeight=560&originWidth=1444&originalType=binary&ratio=2&rotation=0&showTitle=false&size=209003&status=done&style=none&taskId=ubcf9ac16-3752-4a40-bce8-a17009e2947&title=&width=722)
<a name="kTyhQ"></a>
## 切片收缩与扩容
内置的`append()`函数可以添加新的元素到切片的**末尾**，可以接受可变长度的元素，并且可以自动扩容。如果原有数组的长度和容量已经相同，那么在扩容后，长度和容量都会相应增加。
```go
var slice []int
slice = append(slice,0)					// append 单个元素
slice = append(slice,1,2,3)				// append 多个元素
slice = append(slice, []int{4,5,6}...)	// append 切片
```
删除切片的第一个和最后一个元素都非常容易。<br />如果要删除切片中间的某一段或某一个元素，可以借助切片的截取特性，通过截取删除元素前后的切片数组，再使用`append()`函数拼接的方式实现。这种处理方式比较**优雅**，并且**效率高**，因为它不会申请额外的内存空间。
```go
slice := []int{0,1,2,3,4,5,6}
slice := slice[1:]				// 删除第一个元素
slice := slice[:len(slice)-1] 	// 删除最后一个元素

i := int(len(slice)/2）
slice := append(slice[:i],slice[i+1:]...)	// 删除中间某个元素
```
<a name="oUi6S"></a>
# 底层原理
在编译时构建抽象语法树阶段会将切片构建为如下类型。
```go
// Slice contains Type fields specific to slice types.
type Slice struct {
	Elem *Type // element type
}
```
编译时使用`NewSlice()`函数新建一个切片类型，并需要传递切片元素的类型。从中可以看出，切片元素的类型`elem`是在编译期间确定的。
```go
// NewSlice returns the slice Type with element type elem.
func NewSlice(elem *Type) *Type {
	if t := elem.cache.slice; t != nil {
		if t.Elem() != elem {
			base.Fatalf("elem mismatch")
		}
		if elem.HasTParam() != t.HasTParam() || elem.HasShape() != t.HasShape() {
			base.Fatalf("Incorrect HasTParam/HasShape flag for cached slice type")
		}
		return t
	}

	t := newType(TSLICE)
	t.extra = Slice{Elem: elem}
	elem.cache.slice = t
	if elem.HasTParam() {
		t.SetHasTParam(true)
	}
	if elem.HasShape() {
		t.SetHasShape(true)
	}
	return t
}
```
<a name="z3DpN"></a>
## 字面量初始化
当使用形如`[]int{1，2，3}`的字面量创建新的切片时，会创建一个`array`数组（`[3]int{1，2，3}`）存储于**静态区**中，并在**堆区**创建一个新的切片，在程序启动时将静态区的数据复制到堆区，这样可以加快切片的初始化过程。
<a name="hd897"></a>
## make 初始化
对形如`make([]int，3，4)`的初始化切片。在类型检查阶段`typecheck1()`函数中，节点`Node`的`Op`操作为`OMAKESLICE`，并且左节点存储长度为 3，右节点存储容量为 4。<br />**编译时对于字面量的重要优化是判断变量应该被分配到栈中还是应该逃逸到堆区**。

- 如果`make()`函数初始化了一个太大的切片，则该切片会逃逸到堆中。
- 如果分配了一个比较小的切片，则会直接在栈中分配。

此临界值定义在`cmd/compile/internal/gc.maxImplicitStackVarSize`变量中，默认为`64KB`，可以通过指定编译时`smallframes`标识进行更新，因此，`make([]int64，1023)`与`make([]int64，1024)`实现的细节是截然不同的。<br />字面量内存逃逸的核心逻辑位于`cmd/compile/internal/gc/walk.go`，`n.Esc`代表是否判断出变量需要逃逸。

- 如果没有逃逸，那么切片运行时最终会被分配在栈中。
- 如果发生逃逸，那么运行时调用`makesliceXX()`函数会将切片分配在堆中。

当切片的长度和容量小于`int`类型的最大值时，会调用`makeslice()`函数，反之调用`makeslice64()`函数创建切片。<br />`makeslice64()`函数最终也调用了`makeslice()`函数。

1. `makeslice()`函数会先判断要申请的内存大小是否超过了理论上系统可以分配的内存大小，并判断其长度是否小于容量。
2. 再调用`mallocgc()`函数在堆中申请内存，申请的内存大小为类型`大小×容量`。
<a name="kv01U"></a>
## 扩容
切片使用`append()`函数添加元素，但不是使用了`append()`函数就需要进行扩容，切片增加元素后长度超过了现有容量，才需要扩容。
```go
a := make([]int,3,4)
a = append(a,5)		// 不扩容


b := make([]int,3,3)
b = append(b,5) 	// 扩容
```

- 向长度为 3，容量为 4的切片`a`中添加元素后不需要扩容。
- 向长度为 3，容量为 3的切片`b`中添加元素，使用`append(0`函数后，其容量变为了 6。
> 切片扩容的现象说明了 Go 语言并不会在每次`append()`时都进行扩容，也不会每增加一个元素就扩容一次，这是由于扩容常会涉及内存的分配，减慢`append()`的速度。

`append()`函数在运行时调用了`runtime/slice.go`文件下的`growslice()`函数。
```go
// growslice allocates new backing store for a slice.
//
// arguments:
//
//	oldPtr = pointer to the slice's backing array
//	newLen = new length (= oldLen + num)
//	oldCap = original slice's capacity.
//	   num = number of elements being added
//	    et = element type
//
// return values:
//
//	newPtr = pointer to the new backing store
//	newLen = same value as the argument
//	newCap = capacity of the new backing store
//
// Requires that uint(newLen) > uint(oldCap).
// Assumes the original slice length is newLen - num
//
// A new backing store is allocated with space for at least newLen elements.
// Existing entries [0, oldLen) are copied over to the new backing store.
// Added entries [oldLen, newLen) are not initialized by growslice
// (although for pointer-containing element types, they are zeroed). They
// must be initialized by the caller.
// Trailing entries [newLen, newCap) are zeroed.
//
// growslice's odd calling convention makes the generated code that calls
// this function simpler. In particular, it accepts and returns the
// new length so that the old length is not live (does not need to be
// spilled/restored) and the new length is returned (also does not need
// to be spilled/restored).
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	oldLen := newLen - num
	...
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		newcap = newLen
	} else {
		const threshold = 256
		if oldCap < threshold {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < newLen {
				// Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = newLen
			}
		}
	}
	...
	return slice{p, newLen, newcap}
}
```
切片扩容的核心逻辑为：

1. 如果新申请容量（`cap`）大于 2 倍的旧容量（`old.cap`），则最终容量（`newcap`）是新申请的容量（`cap`）。
2. 如果旧切片的长度**小于**`1024`，则最终容量是旧容量的 2 倍，即`newcap=doublecap`。
3. 如果旧切片长度**大于或等于**`1024`，则最终容量从旧容量开始循环增加原来的`1/4`，即`newcap=old.cap`，`for {newcap+=newcap/4}`，直到最终容量**大于或等于**新申请的容量为止，即`newcap ≥ cap`。
4. 如果最终容量计算值溢出，即超过了`int`的最大范围，则最终容量就是新申请容量。

`Growslice()`函数会根据切片的类型，分配不同大小的内存。为了对齐内存，申请的内存可能**大于**`实际的类型大小×容量大小`。<br />如果切片需要扩容，那么最后需要到**堆**区申请内存。<br />要注意的是，切片扩容后返回的地址并不一定和原来的地址相同。因此在使用`append()`函数时，通常会采用`a = append(a，T)`的形式保证其安全。<br />根据`et.ptrdata`是否判断切片类型为指针，执行不同的逻辑。

- 当切片类型**不是指针**时，分配内存后只需要将内存后面的值清空，`memmove(p, old.array, lenmem)`函数用于将旧切片的值赋给新的切片。
- 当切片类型为**指针**，涉及垃圾回收写屏障开启时，对旧切片中指针指向的对象进行标记。
<a name="lLDof"></a>
## 截取
```go
old := make([]int64,3,3)
new := old[1:3]
fmt.Printf("%p, %p",old,new) // 0xc000020090, 0xc000020098
```
根据下标截取的原理，`new`切片截取了`old`切片的第 2、3 号元素，截取后虽然生成了一个新的切片，但是两个切片指向的底层数据源是同一个，可以使用`fmt.Printf`的`%p`格式化打印出变量的地址进行验证。<br />二者的地址正好相差 8 字节，这不是偶然的，而是因为二者指向了相同的数据源，所以刚好相差`int64`的大小。
<a name="NTlBE"></a>
## 完整复制(深拷贝)
深度拷贝是指拷贝切片的底层数组而不是直接部分，因此目标切片不会与源切片共享相同的底层数组。
<a name="v9GRg"></a>
### append()
如果底层数组的容量不足以容纳附加的元素，则`append()`函数将分配一个新的、更大容量的数组保存结果。<br />如果`append()`操作之后创建了一个更大的数组，则新切片将不在与原来的子切片共享相同的底层数组。
```go
// 完整拷贝整个切片
slice1 := []int{1, 2, 3, 4, 5, 6}
slice2 := []int{}
slice2 = append(slice2, slice1...)

fmt.Println(slice1) // => [1 2 3 4 5 6]
fmt.Println(slice2) // => [1 2 3 4 5 6]

// 拷贝一定范围内的值
slice3 := []int{}
slice3 = append(slice3, slice1[3:5]...)
fmt.Println(slice3) // => [4 5]
```
<a name="zpJAM"></a>
### copy()
复制切片时只会复制切片的直接部分，不会改变指向底层的数据源，但有时候希望建一个新的数组，并且与旧数组不共享相同的数据源，这时可以使用`copy()`函数。
```go
func copy(dst, src []T) int
```

- dst 是目标切片，
- src 是源切片。

两个切片必须有相同的元素类型`T`。该函数返回复制的元素数量，取`dst`和`src`长度的最小值。
```go
source := []int{1,2,3,4，5}
target := make([]int,len(source),len(source))
copy(target, source)
```
虽然比较少见，但是切片的数据可以复制到数组中，下例展示了将字节切片的数据存储到字节数组中的情形。
```go
slice := []byte("abcdefg")
var array [4]byte
copy(array[:], slice[:4])
```
如果在复制时，数组的长度与切片的长度不同，例如`copy(arr[:], slice)`，则复制的元素为`len(arr)`与`len(slice)`的较小值。<br />要**深度复制整个切片，必须确保**`**dst**`**具有足够的容量以容纳**`**src**`**的所有元素**。<br />`copy()`函数在运行时主要调用了`memmove()`函数，用于实现内存的复制。如果采用协程调用的方式`go copy(target, source)`或者加入了`race`检测，则会转而调用运行时`slicestringcopy()`或`slicecopy()`函数，进行额外的检查。
<a name="EXSQN"></a>
## 切片的潜在陷阱
正如之前提到过，子切片与原切片**共享底层数组**。<br />因此，当从一个大小为`10MB`的切片`sBig`创建一个大小为`3`字节的子切片`sTiny`时，`sTiny`和`sBig`将引用相同的底层数组。<br />Go 通过垃圾回收机制来自动释放不再被引用的内存的，在这种情况下，即使只需要`3`个字节的`sTiny`，`sBig`仍将继续存在于内存中，因为`sTiny`引用了与`sBig`相同的底层数组。<br />为了解决这个问题，可以进行**深拷贝**，这样`sTiny`不会与`sBig`共享相同的底层数组，就可以被垃圾回收，从而释放内存。
```go
var gopherRegexp = regexp.MustCompile("gopher")

func FindGopher(filename string) []byte {
    // 读取大文件  1,000,000,000 bytes (1GB)
    b, _ := ioutil.ReadFile(filename)
    // 只取一个 6 字节的子切片
    gopherSlice := gopherRegexp.Find(b)
    return gopherSlice
}
```
上面的示例中，读取了一个非常大的文件（`1GB`）并返回了它的一个子切片（仅`6`个字节）。由于 `gopherSlice` 仍然引用与大文件相同的底层数组，这意味着`1GB`的内存即使不再使用也无法被垃圾回收。<br />如果多次调用`FindGopher()`函数，则程序可能会耗尽计算机的所有内存。<br />为了解决这个问题，可以进行**深拷贝**，这样`gopherSlice()`就不会再与巨大的切片共享底层数组。
```go

var gopherRegexp = regexp.MustCompile("gopher")

func FindGopher(filename string) []byte {
    // 读取大文件  1,000,000,000 bytes (1GB)
    b, _ := ioutil.ReadFile(filename)
    // 只取一个 6 字节的子切片
    gopherSlice := make([]byte, len("gopher"))

    // 深度拷贝
    copy(gopherSlice, gopherRegexp.Find(b...)
    return gopherSlice
}
```
这样写的话，Go 的垃圾回收器可以释放大约`1G`的内存。
<a name="iihpt"></a>
## 优化代码性能
<a name="mmsXi"></a>
#### 值拷贝开销
值分配、参数传递、使用`range`关键字循环等，都涉及值拷贝。值大小越大，拷贝的代价就越大，复制`10M`字节所需的时间将比复制`10`字节的时间更长，而拷贝切片时只有直接部分会被复制。
```go
package main

import "fmt"

func main() {
    // Don't do this
    arr := [10]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // arr is copied
    for key, value := range arr {
        fmt.Println(key, value)
    }

    // Do this instead
    for i := 0; i < len(arr); i++ {
        fmt.Println(i, arr[i])
    }
}
```
