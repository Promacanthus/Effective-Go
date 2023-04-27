# Array
几乎所有主流语言都支持数组，它是一片连续的内存区域。<br />Go 语言中的数组与其他语言中的数组有显著不同的特性，例如，其不能进行扩容、在复制和传递时为值复制。通常将数组与切片进行对比。
<a name="CmzHB"></a>
# 声明方式
数组的声明主要有三种方式，如下所示。
```go
var arr1 [3]int
var arr2 = [3]int{1,2,3}	// 声明的同时赋值
arr3 := [...]int{1,2,3}		// 语法糖
```
**数组是内存中一片连续的区域**，需要在初始化时被指定长度，数组的大小取决于数组中存放的元素大小。数组的语法糖，可以不用指定长度，如`arr3 := [...]int{1,2,3}`，这种声明方式在编译时自动推断长度。<br />注意，不同类型的数组之间不能进行比较。

- 数组长度可以通过内置的`len()`函数获取，
- 数组中的元素可以通过下标获取。只能访问数组中已有的元素，如果数组访问越界，则在编译时会报错。
<a name="tGkjB"></a>
# 数组复制
与 C 语言中的数组显著不同的是，Go 语言中的数组在赋值和函数调用时的形参都是**值复制**。
```go
a := [...]int{1,2,3}
b := a
func changeArray(c [3]int)
```
无论是赋值的`b`还是函数调用中的`c`，都是**值复制**，不管是修改`b`还是`c`的值，都不会影响`a`的值，因为他们是完全不同的数组，在内存中的位置都是不同的。
<a name="gSnpt"></a>
# 底层原理
<a name="E5C2e"></a>
## 编译时数组解析
数组形如`[n]T`，在编译时就需要确定其长度和类型。数组在编译时构建抽象语法树阶段的数据类型为`TARRAY`，通过`NewArray()`函数进行创建，`AST`节点的`Op`操作为`OARRAYLIT`。
```go
// NewArray returns a new fixed-length array Type.
func NewArray(elem *Type, bound int64) *Type {
	if bound < 0 {
		base.Fatalf("NewArray: invalid bound %v", bound)
	}
	t := newType(TARRAY)
	t.extra = &Array{Elem: elem, Bound: bound}
	if elem.HasTParam() {
		t.SetHasTParam(true)
	}
	if elem.HasShape() {
		t.SetHasShape(true)
	}
	return t
}
```
`TARRAY`内部的`Array`结构存储了数组中元素的类型及数组的大小。
```go
// Array contains Type fields specific to array types.
type Array struct {
	Elem  *Type // element type
	Bound int64 // number of elements; <0 if unknown yet
}
```
<a name="Uw9vp"></a>
## 数组字面量编译时内存优化
编译时会进行重要的优化。

1. 当数组的长度**小于** 4 时，在运行时数组会被放置在**栈**中，
2. 当数组的长度**大于** 4 时，数组会被放置到内存的静态只读区。
<a name="uRCkZ"></a>
## 数组索引与访问越界
数组访问越界是非常严重的错误，Go 语言中对越界的判断有一部分是在编译期间类型检查阶段完成的，`typecheck1()`函数会对访问数组的索引进行验证。具体的验证逻辑如下：

1. 访问数组的索引是**非整数**时，报错为`non-integer array index%v`。
2. 访问数组的索引是**负数**时，报错为`invalid array index%v (index must be non-negative)`。
3. 访问数组的索引**越界**时，报错为`invalid array index%v (out of bounds for%d-element array)`。

使用未命名常量索引访问数组时，例如`a[3]`，数组的一些简单越界错误能够在编译期间被发现。<br />使用变量索引访问数组或者字符串，编译器无法发现对应的错误，因为变量的值随时可能变化。例如`m:=a[i]`在运行时会检查数组的越界错误。
> Go 语言运行时在发现数组、切片和字符串的越界操作时，会由运行时的`panicIndex()`和`runtime.goPanicIndex()`函数触发程序的运行时错误并导致崩溃。

使用命名常量索引访问数组时，能够在编译时通过优化检测出越界并在运行时报错。
