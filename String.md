# String
<a name="r7QTa"></a>
# 字符串
在编程语言中，字符串是一种重要的数据结构，通常由一系列字符组成。字符串一般有两种类型：

1. 一种编译时指定长度，**不能修改**（字符串常量）。
2. 一种具有动态的长度，**可以修改（字符串变量）**。

Go 语言中字符串**不能被修改**，只能被访问。<br />字符串的结尾有两种方式：

1. 一种是 C 语言中的**隐式**申明，以字符“`\0`”作为终止符。
2. 一种是 Go 语言中的**显式**声明。

Go 语言运行时字符串`string`的表示结构如下。
```go
// StringHeader is the runtime representation of a string.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
//
// In new code, use unsafe.String or unsafe.StringData instead.
type StringHeader struct {
	Data uintptr
	Len  int
}
```

- Data 指向底层的字符数组，
- Len 代表字符串的长度。

字符串在本质上是一串字符数组，每个字符在存储时都对应了一个或多个整数，这涉及字符集的**编码方式**。Go 语言中所有的文件都采用`UTF-8`的编码方式，同时**字符常量**使用`UTF-8`的字符编码集。
> `UTF-8`是一种长度可变的编码方式，可包含世界上大部分的字符。
> 字母都只占据 1 字节，但是特殊的字符（例如大部分中文）会占据 3 字节。

<a name="gk2wH"></a>
# 符文类型
Go 语言的设计者认为用字符（character）表示字符串的组成元素可能产生歧义，因为有些字符非常相似，**这些相似的字符真正的区别在于其编码后的整数是不相同的，**例如：

- 小写拉丁字`a`：被表示为`0x61`
- 带重音符号的`à`：被表示为`0xE0`

因此在 Go 语言中使用符文`rune`类型来表示和区分字符串中的“字符”，`rune`其实是`int32`的别称。当用`range`轮询字符串时，轮询的不再是单字节，而是具体的`rune`。
> Go 的标准库`[unicode/utf8](https://pkg.go.dev/unicode/utf8)`为解释`UTF-8`文本提供了强大的支持，包含了验证、分离、组合`UTF-8`字符的功能。例如`[DecodeRuneInString()](https://pkg.go.dev/unicode/utf8#DecodeRuneInString)`函数返回当前字节之后的符文数及实际的字节长度。

<a name="EGd8D"></a>
# 底层原理
<a name="agKQB"></a>
## 字符串解析
字符串的两种声明方式：
```go
var a string = "hello world"
var b string = `hello world`
```
字符串常量在词法解析阶段最终会被标记成`StringLit`类型的`Token`并被传递到编译的下一个阶段。在语法分析阶段，采取递归下降的方式读取`UTF-8`字符，**单撇号**或**双引号**是字符串的标识。分析的逻辑位于`syntax/scanner.go`文件中。

- 如果在代码中识别到**单撇号**，则调用`rawString()`函数，一直循环向后读取，直到寻找到配对的**单撇号**。
- 如果在代码中识别到**双引号**，则调用`stdString()`函数，
   - 如果出现另一个**双引号**则直接退出；
   - 如果出现`\\`则对后面的字符进行转义。

在**双引号**中不能出现换行符，这是通过对每个字符判断`r=='\\n'`实现的，否则代码在编译时会报错：`newline in string`。
<a name="Dkyml"></a>
## 字符串拼接
在 Go 语言中，可以通过加号操作符（`+`）对字符串进行拼接。由于数字的加法操作也使用加号操作符，因此需要编译时识别具体为何种操作。<br />当加号操作符两边是字符串时，编译时抽象语法树阶段具体操作的`Op`会被解析为`OADDSTR`。对两个字符串**常量**进行拼接时会在语法分析阶段调用`noder.sum`函数。
> 例如对于`"a"+"b"+"c"`的场景，`noder.sum`函数先将所有的字符串常量放到字符串数组中，然后调用`strings.Join()`函数完成对字符串常量数组的拼接。

如果涉及如下字符串**变量**的拼接，那么其拼接操作最终是在运行时完成的。
<a name="wSO4d"></a>
## 运行时字符串拼接
运行时字符串的拼接原理如下图所示：

- 并不是简单地将一个字符串合并到另一个字符串中，
- 而是找到一个更大的空间，并通过内存复制的形式将字符串复制到其中。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1676549767607-602cb6b3-dd95-4733-8745-4ee2049cee06.png#averageHue=%23f7f7f5&clientId=u967d2ce0-ffb5-4&from=paste&height=567&id=uc3130234&originHeight=1134&originWidth=1430&originalType=binary&ratio=2&rotation=0&showTitle=false&size=328017&status=done&style=none&taskId=u1561aeef-b397-4b12-826b-50241e00357&title=&width=715)

- 当拼接后的字符串**小于** 32 字节时，会有一个临时的缓存供其使用。
- 当拼接后的字符串**大于** 32 字节时，**堆区**会开辟一个足够大的内存空间，并将多个字符串存入其中，期间会涉及内存的复制（copy）。
<a name="qrw4P"></a>
## 字符串与字节数组
字节数组与字符串可以相互转换。

1. 字节数组转换为字符串时，在运行时调用`slicebytetostring()`函数。需要注意的是：
   1. 字节数组与字符串的相互转换并不是简单的指针引用，而涉及复制。
   2. 当字符串**大于** 32 字节时，还需要申请**堆内存**，在涉及一些密集的转换场景时，需要评估**性能损耗**。
2. 字符串转换为字节数组时，在运行时调用`stringtoslicebyte()`函数，需要新的足够大小的内存空间。
   1. 当字符串**小于** 32 字节时，可以直接使用缓存`buf`。
   2. 当字符串**大于** 32 字节时，`rawbyteslice()`函数需要向**堆区**申请足够的内存空间。最后使用`copy()`函数完成内存复制。
