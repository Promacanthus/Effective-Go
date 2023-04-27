# map
map（哈希表），是使用频率极高的一种数据结构，原理是将多个键/值对分散存储在`buckets`（桶）中。给定一个键（key），哈希（Hash）算法会计算出键值对存储的位置。
```go
hash = hashfunc(key)
index = hash % array_size
```

1. 通过哈希算法计算键的哈希值，其结果与桶的数量无关。
2. 通过执行取模运算得到`0-array_size-1`之间的`index`序号。

在实践中，通常将`map`看作`o(1)`时间复杂度的操作，通过一个键快速寻找其唯一对应的值（value）。
> 在许多情况下，哈希表的查找速度明显快于一些搜索树形式的数据结构，被广泛用于关联数组、缓存、数据库缓存等场景中。

<a name="c7mrt"></a>
# 哈希碰撞
哈希碰撞（Hash Collision），即不同的键通过哈希函数可能产生相同的哈希值。
> 如果将 2450 个键随机分配到一百万个桶中，则根据概率计算，至少有两个键被分配到同一个桶中的可能性有惊人的 95%。

哈希碰撞导致同一个桶中可能存在多个元素，有多种方式可以避免哈希碰撞，一般有两种主要的策略：拉链法及开放寻址法。
<a name="Lmc4f"></a>
## 拉链法
拉链法将同一个桶中的元素通过链表的形式进行链接，这是一种最简单、最常用的策略。随着桶中元素的增加，可以不断链接新的元素，同时不用预先为元素分配内存。<br />拉链法的不足：

1. 需要存储额外的指针用于链接元素，这增加了整个哈希表的大小
2. 同时由于链表存储的地址不连续，所以无法高效利用 CPU 高速缓存

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1676984498285-93fff25b-d446-4cb8-a5d6-72620f34ced6.png#averageHue=%23d9e5d1&clientId=u4419d645-be20-4&from=paste&height=455&id=ufde62587&originHeight=910&originWidth=1456&originalType=binary&ratio=2&rotation=0&showTitle=false&size=625614&status=done&style=none&taskId=ubdc38d7b-7bbe-44d0-af19-5d8d039e677&title=&width=728)
<a name="kbBor"></a>
## 开放寻址法
开放寻址法（Open Addressing）所有元素都存储在桶的数组中。

- 当必须插入新条目时，将按某种探测策略操作，直到找到未使用的数组插槽为止。
- 当搜索元素时，将按相同顺序扫描存储桶，直到查找到目标记录或找到未使用的插槽为止。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1676985233876-f146a579-0c80-497f-a1c0-bd686b30a2a0.png#averageHue=%23c1dfb6&clientId=u4419d645-be20-4&from=paste&height=627&id=u3719c4c8&originHeight=1254&originWidth=1446&originalType=binary&ratio=2&rotation=0&showTitle=false&size=787000&status=done&style=none&taskId=u559406a0-d6f1-47da-b010-ae60c9412bd&title=&width=723)<br />Go 语言中的哈希表采用的是开放寻址法中的**线性探测（Linear Probing）策略**，按顺序每次探测间隔为 1。<br />由于良好的 CPU 高速缓存利用率和高性能，该算法是现代计算机体系中使用最广泛的结构。
> 接口使用的全局`itab`哈希表采用了开放寻址法中的**二次方探测策略**。

<a name="hdmPv"></a>
# [map](https://github.com/golang/go/blob/master/src/runtime/map.go#L116)
<a name="iKOsx"></a>
## 基本操作
<a name="KjwM0"></a>
### 声明和初始化
```go
var hash map[T]T	// 未初始化，为nil
var hash = make(map[T]T, Number)		// 默认为 0
var hash = map[string]int{ "key" : 1 }	// 字面量初始化
```

- 对于`nil`的`map`进行赋值会触发`panic`
- 对于`nil`的`map`进行访问不会触发`panic`，结果没有意义
<a name="cIN8B"></a>
### 访问
```go
value := hash[key]
value, ok := hash[key]
```

- 返回 2 个参数时，第二个参数表示`key`在`map`中是否存在
<a name="Nme9A"></a>
### 赋值
```go
hash[key] = value	// 赋值
delete(hash, key)	// 删除
```

- 对相同的`key`进行多次删除操作而不会报错
<a name="rJiCf"></a>
### Key的比较性
如果没有办法比较`map`中的`key`是否相同，那么这些`key`就不能作为`map`的`key`，下面简单列出一些基本类型的可比较性。

- **布尔值**、**整数值**、**浮点值**、**复数值**、**字符串值**是可比较的。
- **指针值**是可比较的。如果两个指针值指向相同的变量，或者两个指针的值均为`nil`，则相等。
- **通道值**是可比较的。如果两个通道值是由相同的`make()`函数调用创建的，或者两个值都为`nil`，则相等。
- **接口值**是可比较的。如果两个接口值具有相同的动态类型和相等的动态值，或者两个接口值都为`nil`，则相等。
- 如果**结构**的所有字段都是可比较的，则它们的值是可比较的。
- 如果**数组**元素类型的值可比较，则数组值可比较。如果两个数组对应的元素相等，则它们相等。
- **切片**、**函数**、`map`是不可比较的。
<a name="tGekD"></a>
# 底层实现
在 Go 中表示 map 的底层 struct 是 `hmap`，是 hashmap 的缩写。
```go
type hmap struct {
    count     int	// map中存入元素的个数，调用 len(map) 时直接返回该字段
    flags     uint8	// 状态标记位，通过与定义的枚举值进行&操作可以判断当前是否处于某种状态（如正在写入的状态）
    B         uint8  // 2的B次幂（2^B = bucket size），表示当前 map 中桶（bucket）的数量， B 表示取 hash 后多少位来做 bucket 的分组
    noverflow uint16 // map 中溢出桶的数量，当溢出的桶太多时，map会进行same-size map growth，其实质是避免溢出桶过大导致内存泄露
    hash0     uint32 // 代表生成hash的随机数种子，一般是一个素数

    buckets    unsafe.Pointer // 指向当前map对应的桶的指针，共有 2^B 个 bucket ，如果没有元素存入，这个字段可能为 nil
    oldbuckets unsafe.Pointer // 在扩容期间，将旧的 buckets 数组放在这里，当所有旧桶中的数据都已经转移到了新桶中时，则清空，新 buckets 会是这个的两倍大
    nevacuate  uintptr        // 在扩容时使用，用于标记当前旧桶中地址小于 nevacuate 的数据都已经转移到了新桶中

    extra *mapextra // 存储map中的溢出桶
}
```
B 是 buckets 数组的长度的对数， 即 bucket 数组的长度是 `2^B`。<br />bucket 的本质上是一个指针，指向了一片内存空间，其底层的 struct 是 `bmap`。代表桶的`bmap`结构在运行时只列出了首个字段，即一个固定长度为 8 的数组。此字段顺序存储`key`的哈希值的前 8 位。
```go
// A bucket for a Go map.
type bmap struct {
    tophash [bucketCnt]uint8
}
```
上面的 struct 只是表面的结构，`map`在编译时即确定了`map`中`key`、`value`及桶的大小，动态地创建如下的新结构，因此在运行时仅仅通过指针操作就可以找到特定位置的元素。桶在存储`tophash`字段后，会存储`key`数组及`value`数组。
```go
type bmap struct {
    tophash  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr        // 内存对齐使用，可能不需要
    overflow uintptr        // 当 bucket 的 8 个 key 存满了之后，存储在这里
}
```
`bmap` 就是常说的“**桶**”的底层数据结构，一个桶中可以存放最多 8 个 `key/value`：

- map 使用 hash 函数得到 hash 值决定分配到**哪个桶** 
- 又根据 hash 值的高 8 位来寻找放在**桶的哪个位置** 

具体的 map 的组成结构如下图所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669879367197-779bbe19-f844-49bd-82cb-2a014d86f9b2.png#averageHue=%23f6f3ef&clientId=u08e20f34-a425-4&from=paste&id=ufcd35481&originHeight=818&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=157562&status=done&style=none&taskId=u9125f1b5-a20a-4fdf-84c0-5f371ef4332&title=)
<a name="gM4OK"></a>
## make 初始化
如果使用`make`关键字初始化`map`，则在`typecheck1`类型检查阶段，将节点`Node`的`Op`操作变为`OMAKEMAP`；

- 如果`make`指定了哈希表的长度，则会将长度常量值类型转换为`TINT`
- 如果`make`未指定哈希表的长度，则长度为0（`nodintconst(0)`）
- 如果`make`的第二个参数不是整数，则会在类型检查时报错

在编译时的函数`walk`遍历阶段，`walkexpr()`函数会指定运行时应该调用`runtime.makemap()`函数还是`runtime.makemap64()`函数。<br />`makemap64()`最后也调用了`makemap()`函数，并保证创建`map`的长度不能超过`int`的大小。<br />`makemap()`函数会计算出需要的桶的数量，![](https://cdn.nlark.com/yuque/__latex/77b8f166e0f7aad423b0dccf0eb82394.svg#card=math&code=log%7B2%7DN&id=Hd4x6)，并调用`makeBucketArray()`函数生成桶和溢出桶。<br />如果初始化时生成了溢出桶，则会放置到`map`的`extra`字段。<br />`makeBucketArray()`会为`map`申请内存，需要注意的是，只有`map`的数量大于**24**，才会在初始化时生成溢出桶。<br />溢出桶的大小![](https://cdn.nlark.com/yuque/__latex/8d27e071a7d73bb487f99392caac08a5.svg#card=math&code=2%5E%7Bb-4%7D&id=ySpnJ)，其中，b 初始化时设置的桶的大小。
<a name="W5IaD"></a>
## 字面量初始化
如果`map`采取了字面量初始化的方式，那么它最终仍然需要转换为`make`操作。`map`的长度被自动推断为字面量的长度，其核心逻辑位于`gc/sinit.go`文件的`anylit()`函数中，该函数专门用于处理各种类型的字面量。

- 如果字面量的个数大于**25**，则会构建两个数组专门存储`key`与`value`，在运行时循环添加数据。
- 如果字面量的个数小于**25**，则编译时会通过在运行时初始化时直接添加的方式进行赋值。
<a name="q7V6H"></a>
## map 的存与取
对`map`的访问有两种形式：

1. 一种是返回单个值`v := hash[key]`
2. 一种是返回多个值`v, ok := hash[key]`

Go 没有函数重载的概念，只能在编译时决定返回一个值还是两个值。<br />对于`v := hash[key]`类型的`map`访问，在构建抽象语法树阶段被解析为一个`node`，其中左边的类型为`ONAME`，右边的类型为`OLITERAL`，节点的`Op`操作为`OINDEXMAP`，根据`hash[key]`位于赋值号的**左边**或**右边**，决定要执行**访问**还是**赋值**操作。<br />对于`v, ok := hash[key]`类型的`map`访问则有所不同，在编译时的`Op`操作为`OAS2MAPR`，在运行时最终调用`mapaccess2_XXX()`函数进行`map`访问。
> `mapaccess1()`函数，返回第一个参数值，即`v`
> `mapaccess2()`函数，返回第二个参数值，即`ok`
> 需要注意，如果采用`_, ok := hash[key]`的形式，则不用对第一个参数赋值。

在 map 中**存（赋值）**与**取（访问）**本质上都是在进行一个工作:

1. 查询当前 `k/v` 应该存储的位置
2. 赋值 / 取值

所以理解了 map 中 key 的定位（调用 `[mapaccess2()](https://github.com/golang/go/blob/master/src/runtime/map.go#L456)` 函数），就理解了存取。

- **访问**操作会转化为调用运行时`mapaccess1_XXX()`函数
- **赋值**操作会转换为调用运行时`mapassign_XXX()`函数
> Go 语言编译器会根据`map`中`key`的**类型**和**大小**选择不同的运行时`mapaccess1_XXX()`函数进行加速，这些函数在查找逻辑上都是相同的。

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
    // map 为空，或者元素数为 0，直接返回未找到
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0]), false
        
    }
    // 不支持并发读写
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    // 根据 hash 函数算出 hash 值，注意 key 的类型不同可能使用的 hash 函数也不同
    hash := t.hasher(key, uintptr(h.hash0))
    // 如果 B = 5，那么结果用二进制表示就是 11111 ， 返回的是 B 位全 1 的值
    m := bucketMask(h.B)
    // 根据 hash 的后 B 位，定位在 bucket 数组中的位置
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
    // 当 h.oldbuckets 非空时，说明 map 发生了扩容
    // 这时候，新的 buckets 里可能还没有老的内容
    // 所以一定要在老的里面找，否则有可能发生“消失”的诡异现象
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            // 说明之前只有一半的 bucket，需要除 2
            m >>= 1
        }
        oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
        if !evacuated(oldb) {
            b = oldb
        }
    }
    // tophash 取其高 8bit 的值
    top := tophash(hash)
    // 一个 bucket 在存储满 8 个元素后，就再也放不下了，这时候会创建新的 bucket，挂在原来的 bucket 的 overflow 指针成员上
    // 遍历当前 bucket 的所有链式 bucket
    for ; b != nil; b = b.overflow(t) {
        // 在 bucket 的 8 个位置上查询
        for i := uintptr(0); i < bucketCnt; i++ {
            // 如果找到了相等的 tophash，那说明就是这个 bucket 了
            if b.tophash[i] != top {
                continue
            }
            // 根据内存结构定位 key 的位置
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey {
                k = *((*unsafe.Pointer)(k))
            }
            // 校验找到的 key 是否匹配
            if t.key.equal(key, k) {
                // 定位 v 的位置
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                if t.indirectvalue {
                    v = *((*unsafe.Pointer)(v))
                }
                return v, true
            }
        }
    }

    // 所有 bucket 都没有找到，返回零值和 false
    return unsafe.Pointer(&zeroVal[0]), false
}
```
<a name="NEz32"></a>
### 寻址过程
Go 语言选择将`key`与`value`分开存储而不是以`key/value/key/value`的形式存储，是为了在字节对齐时压缩空间。<br />在进行`hash[key]`的`map`访问操作时，

1. 首先找到桶的位置，如下伪代码所示
```go
hash = hashfunc(key)
index = hash % array_size
```

2. 找到桶的位置后遍历`tophash`数组，如下图所示
3. 如果在数组中找到了相同的`hash`，那么通过指针的寻址操作找到对应的`key`与`value`。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1677210523594-ec7bf15f-1914-4eaf-b56f-5fbf2f0151d0.png#averageHue=%23d0e1ba&clientId=ucf6ad1d3-45c8-4&from=paste&height=498&id=bCUaV&originHeight=996&originWidth=1412&originalType=binary&ratio=2&rotation=0&showTitle=false&size=676141&status=done&style=none&taskId=u8ebb7fc2-0a7a-479c-a1c1-e8e8cde173c&title=&width=706)
```go
// 在运行时，会根据key值及hash种子计算hash值
hash = alg.hash(key, uintptr(h.hash0))

// bucketMask函数计算出当前桶的个数-1
m = bucketMask(h.B)
```
Go 语言采用了一种简单的方式`hash&m`计算出`key`应该位于哪一个桶中。

1. 获取桶的位置后，`tophash(hash)`计算出`hash`的前`8`位。
2. 接着此`hash`挨个与存储在桶中的`tophash`进行对比。
3. 如果有`hash`值相同，则会找到此`hash`值对应的`key`值并判断是否相同。
4. 如果`key`值也相同，则说明查找到了结果，返回`value`。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669880167476-15af2e3d-3697-4daf-b82a-daedd9835d52.png#averageHue=%23d5c56c&clientId=u08e20f34-a425-4&from=paste&height=960&id=u87709f0a&originHeight=1280&originWidth=1005&originalType=binary&ratio=1&rotation=0&showTitle=false&size=155150&status=done&style=none&taskId=ud8c18420-459e-405e-9860-6e8e0700942&title=&width=754)
<a name="fvtpA"></a>
### 赋值过程
和`map`访问的情况类似，赋值操作最终会调用运行时`mapassignXXX()`函数。

- 执行赋值操作时，`map`必须已经进行了初始化，否则在运行时会报错为`assignment to entry in nil map`。
- 由于`map`不支持并发的读写操作，所以每个`map`都有一个`flags`标志位，如果正在执行写入操作，则当前`map`的`hashWriting`标志位会被设置为`1`，
> 在访问时通过检测`hashWriting`即可知道是否有其他协程在访问此`map`，如果是，则报错为`concurrent map writes`。

和访问操作一样，赋值操作时会先计算`key`的`hash`值，标记当前`map`是写入状态。
```go
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
    
	...
    
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}
	hash := t.hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write.
	h.flags ^= hashWriting

    // 如果当前没有桶，则会创建一个新桶，接着找到当前key对应的桶。
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

	...
    bucket := hash & bucketMask(h.B)
    // 如果发现当前的map正在重建，则会优先完成重建过程
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

    ...
	// 最后会计算tophash，开始寻找桶中是否有对应的hash值，
    // 如果找到了，则判断key是否相同，如果相同，则会找到对应的value的位置在后面进行赋值。
    for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
                // 如果没找到tophash，那么赋值操作还会去溢出桶里寻找是否有指定的hash。
                // 如果溢出桶里不存在，则会向第一个空元素中插入数据inserti，insertk会记录此空元素的位置。
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.key.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
    
    // Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
    // 在赋值之前，还要判断map是否需要重建。
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

    // 如果没有问题，就会执行最后的操作，将新的key与value存入数组。
    // 这里需要注意，如果桶中已经没有了空元素，那么需要申请一个新的桶。
    // 新桶一开始来自map中extra字段初始化时存储的多余溢出桶，只有这些多余的溢出桶都用完才会申请新的内存，
    // 溢出桶可以以链表的形式进行延展。溢出桶并不会无限扩展，因为这会带来效率的下降以及可能的内存泄漏。
    if inserti == nil {
		// The current bucket and all the overflow buckets connected to it are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

    ...
}
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1677242167999-97b5b75d-c569-42b7-8e37-5ce7a2ae2437.png#averageHue=%23fbfaf5&clientId=u31e57073-ff00-4&from=paste&height=349&id=ue4f98456&originHeight=698&originWidth=1468&originalType=binary&ratio=2&rotation=0&showTitle=false&size=291533&status=done&style=none&taskId=u23cf26fb-ae92-4192-bccc-86ced12fb28&title=&width=734)
<a name="P9kOH"></a>
### 溢出桶
还有一个溢出桶的概念，在执行`hash[key]=value`赋值操作时，当指定桶中的数据超过 8 个时，并不会直接开辟一个新桶，而是将数据放置到溢出桶中，每个桶的最后都存储了`overflow`，即溢出桶的指针。<br />在正常情况下，数据是很少会跑到溢出桶里面去的。<br />同理，在`map`执行查找操作时，如果`key`的`hash`在指定桶的`tophash`数组中不存在，那么需要遍历溢出桶中的数据。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669880253478-8bed334d-77a0-4072-bd18-1f3968bf904c.png#averageHue=%23fdfbfa&clientId=u08e20f34-a425-4&from=paste&height=374&id=u23431141&originHeight=499&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=87883&status=done&style=none&taskId=uab7461cd-7335-4a7e-93ae-d7ae289fd22&title=&width=810)<br />如果一开始，初始化`map`的数量比较大，桶的数量大于 24 ，则`map`会提前创建好一些溢出桶存储在`extra *mapextra`字段。
```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```

- 这样当出现溢出现象时，可以用提前创建好的桶而不用申请额外的内存空间。
- 只有预分配的溢出桶使用完了，才会新建溢出桶。
<a name="gGz0Z"></a>
## map 的扩容
在 Go 语言中 map 和 slice 一样都是在初始化时首先申请较小的内存空间，在 map 的不断存入的过程中，动态的进行扩容。扩容共有两种，**增量扩容**与**等量扩容**（重新排列并分配内存）。<br />当发生一下两种情况之一时，触发扩容，`map`会进行重建：

1. 负载因子超过阈值，源码里定义的阈值是 6.5。(触发**增量扩容**)
2. 溢出桶的数量过多，即 overflow 的 bucket 数量过多（触发**等量扩容**）：
   1. 当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 (32768) 时，如果 overflow 的 bucket 数量超过 2^B；
   2. 当 B 大于等于 15，也就是 bucket 总数 2^B 大于等于 2^15 (32768) 时，如果 overflow 的 bucket 数量超过 2^15。

重建时需要调用`hashGrow()`函数，

- 如果负载因子超载，则会进行双倍重建。
- 当溢出桶的数量过多时，会进行等量重建。

新桶会存储到`buckets`字段，旧桶会存储到`oldbuckets`字段。`map`中`extra`字段的溢出桶也进行同样的转移。
```go
if h.extra != nil && h.extra.overflow != nil {
    h.extra.oldoverflow = h.extra.overflow
    h.extra.overflow = nil
}
```
要注意的是，这里并没有实际执行将旧桶中的数据转移到新桶的过程。数据转移遵循**写时复制**（copy on write）的规则，只有在真正赋值时，才会选择是否需要进行数据转移，其核心逻辑位于`growWork()`和`evacuate()`函数中。<br />在进行写时复制时，并不是所有的数据都一次性转移，而是只转移当前需要的旧桶中的数据。<br />`bucket := hash&bucketMask(h.B)`得到了当前新桶所在的位置，而要转移的旧桶位于`bucket&h.oldbucketmask()`中。xy [2]evacDst用于存储数据要转移到的新桶的位置。
<a name="Xpspm"></a>
### 增量扩容
负载因子 = 哈希表中的元素数量 / 桶的数量。<br />负载因子的增大，意味着更多的元素会被分配到同一个桶中，此时效率会减慢。
> 如果桶的数量只有 1 个，则此时负载因子到达最大，搜索效率就成了遍历数组。
> Go语言中的负载因子为6.5，

负载因子的最大值为 8，因为一个桶内最多 8 个元素，即当 map 的容量超过（`6.5/8 = 81.25%`）后 map 会进行扩容，增大到原来的 2 倍的大小。旧桶的数据会存到`oldbuckets`字段中，并想办法分散转移到新桶中。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669882480643-bd5b453d-bf53-4cf4-806c-8c43b067c296.png#averageHue=%23f0ebe7&clientId=u08e20f34-a425-4&from=paste&id=uc302e4cf&originHeight=659&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=218531&status=done&style=none&taskId=uf3923879-6e02-42c4-b371-fde3d19e317&title=)<br />在双倍重建中，两个新桶的距离值总是与旧桶的数量值相等。例如，旧桶的数量为2，则转移到新桶的距离也为2。当旧桶中的数据全部转移到新桶中后，旧桶就会被清空。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1677218204264-d5a24659-717c-46fb-90ce-83657fd64085.png#averageHue=%23f9f9f0&clientId=u31e57073-ff00-4&from=paste&height=510&id=u81f8dea2&originHeight=1020&originWidth=1420&originalType=binary&ratio=2&rotation=0&showTitle=false&size=335128&status=done&style=none&taskId=u5a1966b6-8732-4548-85bf-f0087e1d25e&title=&width=710)<br />双倍重建时，还需要解决旧桶中的数据要转移到某一个新桶中的问题。<br />其中有一个非常重要的原则：如果数据的`hash&bucketMask`**小于或等于**旧桶的大小，则此数据必须转移到和旧桶位置完全对应的新桶中去，理由是当前`key`所在新桶的序号与旧桶是完全相同的。
<a name="l5lnY"></a>
### 等量扩容
map 的重建的另一种情况，即溢出桶的数量太多（说明出现哈希碰撞的次数太多），需要 rehash，这时 map 只会新建和原来相同大小的桶，只是将key/value 进行一次 rebalance，map 的总容量不变，目的是防止溢出桶的数量缓慢增长导致的内存泄露。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669882506433-d70310dc-9244-4525-975c-9655be20ee4f.png#averageHue=%23f1ece7&clientId=u08e20f34-a425-4&from=paste&id=u54772dc1&originHeight=510&originWidth=836&originalType=binary&ratio=1&rotation=0&showTitle=false&size=138254&status=done&style=none&taskId=u316401ba-1574-4d60-aa97-746bcc2f43d&title=)<br />等量重建，进行简单的直接转移即可。
<a name="MUTQI"></a>
## map 的删除
当进行`map`的`delete`操作时，和赋值操作类似，

1. `delete`操作会根据`key`找到指定的桶
2. 如果存在指定的`key`，那么就释放掉`key`与`value`引用的内存
3. 同时`tophash`中的指定位置会存储`emptyOne`，代表当前位置是空的

![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1677218248688-0537dec5-8598-4cb6-8bfe-5944c08481cd.png#averageHue=%23eae8c5&clientId=u31e57073-ff00-4&from=paste&height=308&id=u09a7f996&originHeight=616&originWidth=1430&originalType=binary&ratio=2&rotation=0&showTitle=false&size=543574&status=done&style=none&taskId=u79584149-0ce2-4ba9-a132-849c9ff3a98&title=&width=715)<br />同时，删除操作会探测当前要删除的元素之后是否都是空的。如果是，则`tophash`会存储为`emptyRest`，并循环向上检查前一个元素是否为空。这样做的好处是在做查找操作时，遇到`emptyRest`可以直接退出，因为后面的元素都是空的。
<a name="QjBlz"></a>
## map 有序性
在 Go 语言中 map 是无序的，准确的说是无法严格保证顺序的，因为 map 在扩容后，可能会将部分 key 移至新内存，由于在扩容搬移数据过程中，并未记录原数据位置， 并且在 Go 的数据结构中也并未保存数据的顺序，所以这一部分在扩容后就已经是无序的了。<br />虽然遍历的过程是按顺序遍历内存地址，同时按顺序遍历内存地址中的 key。但是，即使 map 不进行扩容，在多次遍历时也是无序的，因为 Go 语音设计时故意加上随机的元素，将遍历 map 的顺序随机化，用来防止使用者用来顺序遍历。<br />依赖 map 的顺序进行遍历，这是有风险的代码，在 Go 的严格语法规则下，是坚决不提倡的。所以在使用 map 时一定要记得是无序的，不要依赖其顺序。
<a name="tXCPb"></a>
## map 的并发
在 Go 语言中 `map` 不是并发安全的数据结构：

- 当几个 `goruotine` 同时对一个`map`进行**写操作**时，就会出现并发写问题：`fatal error: concurrent map writes`
- 只支持多个 `goroutine` 对一个`map`进行**读操作**
```go
// 不支持并发读写
if h.flags&hashWriting != 0 {
	throw("concurrent map read and map write")
}
```
map 不支持并发安全的主要原因**成本与效益**。官方答复原因如下：

- **典型使用场景**：map 的典型使用场景是不需要从多个 goroutine 中进行安全访问。
- **非典型场景**（需要原子操作）：map 可能是一些更大的数据结构或已经同步的计算的一部分。
- **性能场景**：若是只是为少数程序增加安全性，导致 map 所有的操作都要处理 mutex，将会降低大多数程序的性能。

为什么 map 并发冲突后要选择让程序 crash 崩溃掉，是 Go 官方出于权衡风险和 map 使用复杂度，首先 map 在官方中就明确表示不支持并发读写， 所以对 map 进行并发读写操作本身就是不正确的。

1. 场景假设一：如果 map 选择在写入或者读取时增加 error 返回值，会导致程序在使用 map 时就无法像现在一样，需要额外的捕获并判断 err。
2. 场景假设二：如果 map 选择 panic（可被 recover），此时如果出现并发写入数据的场景，就会导致走进 recover 中，如果没有对这种场景进行特殊处理，就会导致 map 中存在脏数据，此时程序在使用 map 时就会引发不可预知的错误，此时排查起来也是很难找到问题的根因的。

所以明确的抛出 crash 崩溃异常，使得风险被提前暴露，可以明确的定位到问题点。<br />综上在使用 map 时，要严格保障其是在单线程内使用的，如果有多线程场景，建议使用`sync.Map` 。
<a name="cP3gd"></a>
# [sync.Map](https://github.com/golang/go/blob/master/src/sync/map.go#L33)
原生 map 是不支持并发读写的，高并发的场景解决的办法有两个：

- `sync.Mutex` / `sync.RWMutex`：方案简单，但性能不会太高。
- 使用并发安全的字典类型 `sync.Map`（Go1.9 引入）：方案优雅实用。

`sync.Map` 如何在 lock free 的前提下，保证足够高的性能呢？除了使用锁，原子操作（atomic）也可以达到类似并发安全的目的。`sync.Map`的设计非常巧妙，充分利用了`atmoic`和`mutex`的配合。核心思想就是，**尽量使用原子操作，最大程度上减少了锁的使用，从而接近了“lock free”的效果。**

- 使用了两个原生的 map 作为存储介质，分别是 read map（只读字典） 和 dirty map（脏字典）。
- 只读字典使用 `atomic.Value` 来承载，保证原子性和高性能；脏字典则需要用互斥锁来保护，保证互斥。
- 只 read 和 dirty 中的键值对集合并不是实时同步的，它们在某些时间段内可能会有不同。
- 无论是 read 还是 dirty，本质上都是`map[any]*entry`类型，这里的 entry 就是 map 的 value 的容器。
- entry 的本质，是一层封装，可以表示具体值的指针，也可以表示 key 已删除的状态（即逻辑假删除）。通过这种设计，规避了原生 map 无法并发安全 delete 的问题，同时在变更某个键所对应的值时，就可以使用原子操作。
```go
type Map struct {
    mu sync.Mutex

    // read map 是被 atomic 包托管的，这意味着它本身 Load 是并发安全的（但是它的 Store 操作需要锁 mu 的保护）
    // read map 中的 entries 可以安全地并发更新，但是对于 expunged entry，在更新前需要经它 unexpunge 化并存入 dirty
    //（这句话，在 Store 方法的第一种特殊情况中，使用 e.unexpungeLocked 处有所体现）
    read atomic.Value // readOnly

    // 关于 dirty map 必须要在锁 mu 的保护下，进行操作，它仅仅存储 non-expunged entries
    // 如果一个 expunged entries 需要存入 dirty，需要先进行 unexpunged 化处理
    // 如果 dirty map 是 nil 的，则对 dirty map 的写入之前，需要先根据 read map 对 dirty map 进行浅拷贝初始化
    dirty map[any]*entry

    // 每当读取的是时候，read 中不存在，需要去 dirty 查看，miss 自增，
    // 到一定程度会触发 dirty=>read 升级转储，升级完毕之后:
    // - dirty = nil
    // - miss = 0 
    // - read.amended = false
    misses int
}

// 这是一个被原子包 atomic.Value 托管了的结构，内部仍然是一个 map[any]*entry
// 以及一个 amended 标记位，如果为真，则说明 dirty 中存在新增的 key，还没升级转储，即不在 read 中
type readOnly struct {
    m       map[any]*entry
    amended bool // true if the dirty map contains some key not in m.
}

// expunged 是一个任意的指针，用于标记已经从 dirty 中删除的 entry
var expunged = unsafe.Pointer(new(any))

// 这是一个容器，可以存储任意的东西，因为成员 p 是 unsafe.Pointer
// sync.Map 中的值都不是直接存入 map 的，都是在 entry 的包裹下存入的
type entry struct {
    // entry 的 p 可能的状态：
    // 	e.p == nil：entry 已经被标记删除，不过此时还未经过 read=>dirty 重塑，此时可能仍然属于 dirty（如果 dirty 非 nil）
    // 	e.p == expunged：entry 已经被标记删除，经过 read=>dirty 重塑，不属于 dirty，仅属于 read，下一次 dirty=>read 升级，会被彻底清理
    // 	e.p == 普通指针：此时 entry 是一个不同的存在状态，属于 read，如果 dirty 非 nil，也属于 dirty
    p unsafe.Pointer // *interface{}
}
```
<a name="SMB5U"></a>
## 架构设计
![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669892203961-a260211b-009a-48c7-8b19-514b3d7d5c5a.png#averageHue=%23e0e9c3&clientId=u5819dd9b-7030-4&from=paste&id=uc97de4f0&originHeight=474&originWidth=727&originalType=binary&ratio=1&rotation=0&showTitle=false&size=56483&status=done&style=none&taskId=ucef916ed-e810-49eb-aa17-8b3ec5313d5&title=)

- read map 是`atomic`包托管，主要负责高性能，但是无法保证拥有全量的key（因为新增 key 会先加到 dirty 中），某种程度上，类似于一个 key 的快照。
- dirty map 拥有全量的 key，当 Store 操作要新增一个 key 时，会增加到 dirty 中。

查找指定 key 时：

1. 先去 read 中寻找，不需要锁定互斥锁。
2. 当 read 中没有，但 dirty 中可能会有时，才会在锁的保护下去访问 dirty。

存储键值对时：

1. 只要 read 中已存有这个 key，并且该键值对未被标记为“`expunged`”，就会把新值存到里面并直接返回，这种情况下也不需要用到锁。

`expunged`和`nil`，都表示标记删除，但有区别的：

- `expunged`是 read **独有**的
- `nil`则是 read 和 dirty **共有**的
<a name="Gucv4"></a>
### read 和 dirty 互转
read 和 dirty 是一直在动态变化的，可能存在重叠（分为`nil`和`normal`），也可能是某一方为空。

- **dirty=>read**：随着 load 的 miss 不断自增，达到阈值后触发升级转储，完毕之后：
   - `dirty = nil`
   - `miss = 0`
   - `read.amended = false`
- **read=>dirty**：当有 read 中不存在的新 key 需要增加且 read 和 dirty 一致时，触发重塑，且`read.amended=true`（然后再在 dirty 新增）。重塑的过程：
   1. 将 nil 状态的 entry，全部挤压到 expunged 状态中，
   2. 将非 expunged 的 entry 浅拷贝到 dirty 中，避免 read 的 key 无限的膨胀（存在大量逻辑删除的 key）。
   3. 在 dirty 再次升级为 read 的时候，这些逻辑删除的 key 就可以一次性丢弃释放了（因为是直接覆盖上去）

![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669894093966-11df8870-3684-468a-becc-6a5c8c44d7c5.png#averageHue=%23f8f8f8&clientId=u5819dd9b-7030-4&from=paste&id=u1ce4a24b&originHeight=316&originWidth=510&originalType=binary&ratio=1&rotation=0&showTitle=false&size=37398&status=done&style=none&taskId=u9894ec57-b452-4cd6-b6ac-4f93ce7998b&title=)
<a name="yjCIF"></a>
### read 从何而来，存在的意义

- read 是 dirty 升级而来，利用`atomic.Store`一次性覆盖，而不是一点点的 set 操作出来的。所以，read 更像是一个快照，read 中 key 的集合不能被改变（注意，这里说的 read 的 key 不可改变，value 是可以通过 CAS 来进行更改的），所以其中的键的集合有时候可能是不全的。
- dirty 中的键值对集合总是完全的，但是其中不会包含 expunged 的键值对。
- read 存在价值，在于加速读性能（通过原子操作避免了锁）。
<a name="XGjJr"></a>
### entry 的 p 可能的状态

- `e.p==nil`：entry 已经被标记删除，不过此时还未经过`read=>dirty`重塑，此时可能仍然属于dirty（如果 dirty 非 nil）
- `e.p==expunged`：entry 已经被标记删除，经过`read=>dirty`重塑，不属于 dirty，仅仅属于read，下一次`dirty=>read`升级，会被彻底清理（因为升级的操作是直接覆盖，read 中的expunged 会被自动释放回收）
- `e.p==普通指针`：entry 是一个普通的存在状态，属于 read，如果 dirty 非 nil，也属于 dirty，对应架构图中的 normal 状态。
<a name="oQxtq"></a>
### 删除操作时`e.p`设置成 nil 还是 expunged

- 如果 key 不在 read 中，但是在 dirty 中，则直接 delete。
- 如果 key 在 read 中，则逻辑删除，`e.p`赋值为`nil`(后续重塑时，`nil`会变成`expunged`)
<a name="CggbQ"></a>
### e.p 由 nil 变成 expunged 的时机

- read=>dirty 重塑的时候，此时 read 中仍然是 nil 的，会变成 expunged，表示这部分 key 等待被最终丢弃（expunged 是最终态，等待被丢弃，除非又出现了重新 store 的情况）
- 最终丢弃的时机：就是 dirty=>read 升级的时候，dirty 的直接粗暴覆盖，会使得 read 中的所有成员都被丢弃，包括 expunged。
<a name="oGaRQ"></a>
### 设计 expunged 的意义
expunged 是有存在意义的，它作为删除的最终状态（待释放），这样 nil 就可以作为一种中间状态。如果仅仅使用 nil，那么，在 read=>dirty 重塑的时候，可能会出现如下的情况：

- 如果 nil 在 read 浅拷贝至 dirty 的时候仍然保留 entry 的指针（即拷贝完成后，对应键值下 read 和dirty 中都有对应键下 entry e 的指针，且`e.p=nil`）那么之后在 dirty=>read 升级 key 的时候对应entry 的指针仍然会保留。那么最终合集会越来越大，存在大量 nil 的状态，永远无法得到清理的机会。
- 如果 nil 在 read 浅拷贝时不进入 dirty，那么之后 Store 某个 Key 键的时候，可能会出现 read 和 dirty 不同步的情况，即此时 read 中包含 dirty 不包含的键，那么之后用 dirty 替换 read 的时候就会出现数据丢失的问题。
- 如果 nil 在 read 浅拷贝时直接把 read 中对应键删除（从而避免了不同步的问题），但这又必须对read 加锁，违背了 read 读写不加锁的初衷。

综上，为了保证 read 作为快照的性质（不能单独删除或新增 key），同时要避免 map 中 nil 的 key 不断膨胀等多个前提要求，才设计成了 expungd 的状态。
<a name="xB3lL"></a>
###  entry 状态图
![image.png](https://cdn.nlark.com/yuque/0/2022/png/362867/1669895072630-88c2cd50-9e28-47f5-a2b9-99fed7fcd679.png#averageHue=%23f9f9f9&clientId=u5819dd9b-7030-4&from=paste&id=u3a4f246a&originHeight=258&originWidth=738&originalType=binary&ratio=1&rotation=0&showTitle=false&size=39604&status=done&style=none&taskId=u38f905ab-fa13-45a8-b09b-474cb1843ea&title=)
<a name="fXsXY"></a>
## Store（对应 C/U）
```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
    // 首先把 readonly 字段原子地取出来
    // 如果 key 在 readonly 里面，则先取出 key 对应的 entry，
    // 然后尝试对这个 entry 存入 value 的指针
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 如果 readonly 里面不存在 key 或者是对应的 key 是被擦除掉了的，则继续。。。
    m.mu.Lock() // 上锁

    // 锁的惯用模式：再次检查 readonly，防止在上锁前的时间缝隙出现存储
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        // 这里有两种情况：
        // 1. 上面的时间缝隙里面，出现了 key 的存储过程（可能是 normal 值，也可能是 expunge 值）
        //    此时先校验 e.p，如果是普通值，说明 read 和 dirty 里都有相同的 entry，则直接设置 entry
        //    如果是 expunge 值，则说明 dirty 里面已经不存在 key 了，需要先在 dirty 里面种上 key，然后设置 entry
        // 2. 本来 read 里面就存在，只不过对应的 entry 是 expunge 的状态
        //    这种情况和上面的擦除情况一样，说明 dirty 里面已经不存在 key 了，需要先在 dirty 里面存储 key，然后设置 entry
        if e.unexpungeLocked() {
           // 进入这里说明 entry 之前是 expunged 状态，
           // 也就是说，此时 dirty 是非 nil，并且此 entry 不在 dirty 中 
           // 通过 CAS 将 entry 的 p 改为 nil，并且在 dirty 中也写入
           m.dirty[key] = e
       }
        e.storeLocked(&value) // 将 value 存入容器 e
    } else if e, ok := m.dirty[key]; ok {
        // readonly 里面不存在，则查看 dirty 里面是否存在
        // 如果 dirty 里面存在，则直接设置 dirty 的对应 key
        e.storeLocked(&value)
    } else {
        // dirty 里面也不存在（或者 dirty 为 nil），则应该先设置在 dirty 里面
        // 此时要检查 read.amended，如果为假（标识 dirty 中没有自己独有的 key or 两者均是初始化状态）
        // 此时要在 dirty 里面设置新的 key，需要确保 dirty 是初始化的且需要设置 amended 为 true（表示自此 dirty 多出了一些独有 key）
        if !read.amended {
           // 表示第一次向 ditry 中添加新的 key，同时修改 readonly 中 amended 为 true
           m.dirtyLocked()
           m.read.Store(readOnly{m: read.m, amended: true})
       }
        m.dirty[key] = newEntry(value)
    }

    // 解锁
    m.mu.Unlock()
}

// 这是一个自旋乐观锁：只有 key 是非 expunged 的情况下，会得到 set 操作
func (e *entry) tryStore(i *interface{}) bool {
    for {
        p := atomic.LoadPointer(&e.p)
        // 如果 p 是 expunged 就不可以 set 了
        // 因为 expunged 状态是 read 独有的，
        // 这种情况下说明这个 key 已经删除（并且发生过了 read=>dirty 重塑过）了
        // 此时要新增只能在 dirty 中，不能在 read 中
        if p == expunged {
           return false
       }
        // 如果非 expunged，则说明是 normal 的 entry 或者 nil 的 entry，可以直接替换
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
           return true
       }
    }
}

// 利用 CAS，如果 e.p 是 expunged，则将 e.p 置为 nil，从而保证她是 read 和 dirty 共有的
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// 真正的 set 操作，从这里也可以看出来2点：
// - 1、是 set 是原子的 
// - 2、是封装的过程
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

// 利用 read 重塑 dirty
// 如果 dirty 为 nil，则利用当前的 read 来初始化 dirty（包括 read 本身也为空的情况）
// 此函数是在锁的保护下进行，所以不用担心出现不一致
func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    // 经过这么一轮操作:
    // dirty 里面存储了全部的非 expunged 的 entry
    // read 里面存储了 dirty 的全集，以及所有 expunged 的 entry
    // 且 read 中不存在 e.p == nil 的 entry（已经被转成了expunged）
    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[any]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() { // 只有非擦除的 key，能够重塑到 dirty 里面
           m.dirty[k] = e
       }
    }
}

// 利用乐观自旋锁，
// 如果 e.p 是 nil，尽量将 e.p 置为 expunged
// 返回最终 e.p 是否是 expunged
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
           return true
       }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}
```
<a name="s2BWd"></a>
## Load（对应 R）
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 把 readonly 字段原子地取出来
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]

    // 如果 readonly 没找到，且 dirty 包含了 read 没有的 key，则尝试去 dirty 里面找
    if !ok && read.amended {
       m.mu.Lock()
       // 锁的惯用套路
       read, _ = m.read.Load().(readOnly)
       e, ok = read.m[key]
       if !ok && read.amended {
          e, ok = m.dirty[key]
          // 记录 miss 次数，并在满足阈值后，触发 dirty=>read 的升级
          m.missLocked()
      }
       m.mu.Unlock()
   }

    // readonly 和 dirty 的 key 列表，都没找到，返回 nil
    if !ok {
       return nil, false
   }

    // 找到了对应 entry，随即取出对应的值
    return e.load()
}

// 自增 miss 计数器
// 如果增加到一定程度，dirty 会升级成为 readonly
// - dirty=nil 
// - read.amended=false
// - misses=0
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
       return
   }
    // 直接用 dirty 覆盖到了 read 上
    // 也就是意味着 dirty 的值是必然是 read 的父集合，
    // 当然这不包括 read 中的 expunged entry）
    m.read.Store(readOnly{m: m.dirty}) // 这里有一个隐含操作，read.amended 再次变成 false
    m.dirty = nil
    m.misses = 0
}

// entry 是一个容器，从 entry 里面取出实际存储的值（以指针提取的方式）
func (e *entry) load() (value interface{}, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == nil || p == expunged {
       return nil, false
   }
    return *(*interface{})(p), true
}
```
<a name="hyO5Z"></a>
## Delete（对应 D）
```go

// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
    m.LoadAndDelete(key)
}

// 删除的逻辑和 Load 的逻辑，基本上是一致的
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
       m.mu.Lock()
       read, _ = m.read.Load().(readOnly)
       e, ok = read.m[key]
       if !ok && read.amended {
          e, ok = m.dirty[key]
          delete(m.dirty, key)
          m.missLocked()
      }
       m.mu.Unlock()
   }
    if ok {
       return e.delete()
   }
    return nil, false
}

// 如果 e.p == expunged 或者 nil，则返回 false
// 否则，设置 e.p = nil，返回删除的值得指针
func (e *entry) delete() (value interface{}, ok bool) {
    for {
       p := atomic.LoadPointer(&e.p)
       if p == nil || p == expunged {
          return nil, false
      }
       if atomic.CompareAndSwapPointer(&e.p, p, nil) {
          return *(*interface{})(p), true
      }
   }
}

```

