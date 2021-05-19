```json
{
  "date": "2021.05.19 17:50",
  "tags": ["golang","map","float"],
  "description":"记录一下map的底层实现，包括内存模型，哈希函数，key定位过程，map的赋值、遍历、删除、扩容等"
}
```

[什么是map](#jump1)

[map的底层实现](#jump2)

&emsp; &emsp; [map内存模型](#jump3)

&emsp; &emsp; [创建map](#jump4)

&emsp; &emsp; [哈希函数](#jump5)

&emsp; &emsp; [key定位过程](#jump6)

&emsp; &emsp; [map的赋值过程](#jump7)

&emsp; &emsp; [map的遍历过程](#jump8)

&emsp; &emsp; [map的删除过程](#jump9)

&emsp; &emsp; [map的扩容过程](#jump10)

# <span id="jump1">什么是map</span>

map 的设计也被称为 “The dictionary problem”，它的任务是设计一种数据结构用来维护一个集合的数据，并且可以同时对集合进行增删查改的操作。
最主要的数据结构有两种：哈希查找表（Hash table）、搜索树（Search tree）。

哈希查找表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）。这样，开销主要在哈希函数的计算以及数组的常数访问时间。
在很多场景下，哈希查找表的性能很高。 哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：链表法和开放地址法。
链表法将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。开放地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。

搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树。

自平衡搜索树法的最差搜索效率是 O(logN)，而哈希查找表最差是 O(N)。当然，哈希查找表的平均查找效率是 O(1)，如果哈希函数设计的很好，最坏的情况基本不会出现。
还有一点，遍历自平衡搜索树，返回的 key 序列，一般会按照从小到大的顺序；而哈希查找表则是乱序的。

# <span id="jump2">map的底层实现</span>

GO的版本
```go
go version go1.16.2 darwin/amd64
```

## <span id="jump3">map内存模型</span>
在源码中，表示 map 的结构体是 hmap，它是 hashmap 的“缩写”：
```go
// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    count     int // # 元素个数，调用 len(map) 时，直接返回此值 Must be first (used by len() builtin)
    flags     uint8
    B         uint8  // buckets 的对数 log_2
    noverflow uint16 // overflow 的 bucket 近似数
    hash0     uint32 // hash seed 计算 key 的哈希的时候会传入哈希函数
    
    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
    
    extra *mapextra // optional fields
}
```
说明一下，B 是 buckets 数组的长度的对数，也就是说 buckets 数组的长度就是 2^B。bucket 里面存储了 key 和 value，后面会再讲。

buckets 是一个指针，最终它指向的是一个结构体：
```go
type bmap struct {
    tophash [bucketCnt]uint8
}
```
但这只是表面(src/runtime/hashmap.go)的结构，编译期间会给它加料，动态地创建一个新的结构：
```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```
bmap 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。
在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）。

来一个整体的图：
<img src="./images/hmap.png" alt="hmap" style="zoom:40%;" />
当 map 的 key 和 value 都不是指针，并且 size 都小于 128 字节的情况下，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。但是，我们看 bmap 其实有一个 overflow 的字段，是指针类型的，破坏了 bmap 不含指针的设想，这时会把 overflow 移动到 extra 字段来。
```go
type mapextra struct {
    // overflow[0] contains overflow buckets for hmap.buckets.
    // overflow[1] contains overflow buckets for hmap.oldbuckets.
    overflow [2]*[]*bmap
    // nextOverflow 包含空闲的 overflow bucket，这是预分配的 bucket
    nextOverflow *bmap
}
```
bmap 是存放 k-v 的地方，看 bmap 的内部组成。
<img src="./images/bmap.png" alt="bmap" style="zoom:40%;" />
上图就是 bucket 的内存模型，HOB Hash 指的就是 top hash。 注意到 key 和 value 是各自放在一起的，并不是 key/value/key/value/... 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

例如，有这样一个类型的 map：map[int64]int8

如果按照 key/value/key/value/... 这样的模式存储，那在每一个 key/value 对之后都要额外 padding 7 个字节；而将所有的 key，value 分别绑定到一起，这种形式 key/key/.../value/value/...，则只需要在最后添加 padding。

每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来。
## <span id="jump4">创建map</span>
从语法层面上来说，创建 map 很简单：
```go
ageMp := make(map[string]int)
// 指定 map 长度
ageMp := make(map[string]int, 8)
// ageMp 为 nil，不能向其添加元素，会直接panic
var ageMp map[string]int
```
实际上底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等等。
```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
    // 省略各种条件检查...
    // 找到一个 B，使得 map 的装载因子在正常范围内
    B := uint8(0)
    for ; overLoadFactor(hint, B); B++ {
    }
    // 初始化 hash table
    // 如果 B 等于 0，那么 buckets 就会在赋值的时候再分配
    // 如果长度比较大，分配内存会花费长一点
    buckets := bucket
    var extra *mapextra
    if B != 0 {
        var nextOverflow *bmap
        buckets, nextOverflow = makeBucketArray(t, B)
        if nextOverflow != nil {
            extra = new(mapextra)
            extra.nextOverflow = nextOverflow
        }
    }
    // 初始化 hamp
    if h == nil {
        h = (*hmap)(newobject(t.hmap))
    }
    h.count = 0
    h.B = B
    h.extra = extra
    h.flags = 0
    h.hash0 = fastrand()
    h.buckets = buckets
    h.oldbuckets = nil
    h.nevacuate = 0
    h.noverflow = 0
    return h
}
```
【引申1】slice 和 map 分别作为函数参数时有什么区别？

注意，这个函数返回的结果：*hmap，它是一个指针，而我们之前讲过的 makeslice 函数返回的是 Slice 结构体：
```go
func makeslice(et *_type, len, cap int) slice
```
回顾一下 slice 的结构体定义：
```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int // 长度 
    cap   int // 容量
}
```
结构体内部包含底层的数据指针。

makemap 和 makeslice 的区别，带来一个不同点：当 map 和 slice 作为函数参数时，在函数参数内部对 map 的操作会影响 map 自身；而对 slice 却不会（之前讲 slice 的文章里有讲过）。

主要原因：一个是指针（*hmap），一个是结构体（slice）。Go 语言中的函数传参都是值传递，在函数内部，参数会被 copy 到本地。*hmap指针 copy 完之后，仍然指向同一个 map，因此函数内部对 map 的操作会影响实参。而 slice 被 copy 后，会成为一个新的 slice，对它进行的操作不会影响到实参。

## <span id="jump5">哈希函数</span>
map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 alginit() 中完成，位于路径：src/runtime/alg.go 下。
```text
hash 函数，有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种；非加密型的一般就是查找。在 map 的应用场景中，用的是查找。选择 hash 函数主要考察的是两点：性能、碰撞概率。
```
之前我们讲过，表示类型的结构体：
```go
type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldalign uint8
    kind       uint8
    alg        *typeAlg
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```
其中 alg 字段就和哈希相关，它是指向如下结构体的指针：
```go
// src/runtime/alg.go
type typeAlg struct {
    // (ptr to object, seed) -> hash
    hash func(unsafe.Pointer, uintptr) uintptr
    // (ptr to object A, ptr to object B) -> ==?
    equal func(unsafe.Pointer, unsafe.Pointer) bool
}
```
typeAlg 包含两个函数，hash 函数计算类型的哈希值，而 equal 函数则计算两个类型是否“哈希相等”。

对于 string 类型，它的 hash、equal 函数如下：
```go
func strhash(a unsafe.Pointer, h uintptr) uintptr {
    x := (*stringStruct)(a)
    return memhash(x.str, h, uintptr(x.len))
}
func strequal(p, q unsafe.Pointer) bool {
    return *(*string)(p) == *(*string)(q)
}
```
根据 key 的类型，_type 结构体的 alg 字段会被设置对应类型的 hash 和 equal 函数。
## <span id="jump6">key定位过程</span>
key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。

例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：
```text
 10010111 | 000011110110110010001111001010100010010110010101010 │ 01010
```
用最后的 5 个 bit 位，也就是 01010，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。
<img src="./images/key.png" alt="key" style="zoom:40%;" />
上图中，假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 00110，找到对应的 6 号 bucket，使用高 8 位 10010111，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key，找到了 2 号槽位，这样整个查找过程就结束了。

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。

查找某个 key 的底层函数是 mapacess 系列函数，函数的作用类似，区别在下一节会讲到。这里我们直接看 mapacess1 函数：
```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // ……
    // 如果 h 什么都没有，返回零值
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0])
    }
    // 写和读冲突
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    // 不同类型 key 使用的 hash 算法在编译期确定
    alg := t.key.alg
    // 计算哈希值，并且加入 hash0 引入随机性
    hash := alg.hash(key, uintptr(h.hash0))
    // 比如 B=5，那 m 就是31，二进制是全 1
    // 求 bucket num 时，将 hash 与 m 相与，
    // 达到 bucket num 由 hash 的低 8 位决定的效果
    m := uintptr(1)<<h.B - 1
    // b 就是 bucket 的地址
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // oldbuckets 不为 nil，说明发生了扩容
    if c := h.oldbuckets; c != nil {
        // 如果不是同 size 扩容（看后面扩容的内容）
        // 对应条件 1 的解决方案
        if !h.sameSizeGrow() {
            // 新 bucket 数量是老的 2 倍
            m >>= 1
        }
        // 求出 key 在老的 map 中的 bucket 位置
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 如果 oldb 没有搬迁到新的 bucket
        // 那就在老的 bucket 中寻找
        if !evacuated(oldb) {
            b = oldb
        }
    }
    // 计算出高 8 位的 hash
    // 相当于右移 56 位，只取高8位
    top := uint8(hash >> (sys.PtrSize*8 - 8))
    // 增加一个 minTopHash
    if top < minTopHash {
        top += minTopHash
    }
    for {
        // 遍历 8 个 bucket
        for i := uintptr(0); i < bucketCnt; i++ {
            // tophash 不匹配，继续
            if b.tophash[i] != top {
                continue
            }
            // tophash 匹配，定位到 key 的位置
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // key 是指针
            if t.indirectkey {
                // 解引用
                k = *((*unsafe.Pointer)(k))
            }
            // 如果 key 相等
            if alg.equal(key, k) {
                // 定位到 value 的位置
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                // value 解引用
                if t.indirectvalue {
                    v = *((*unsafe.Pointer)(v))
                }
                return v
            }
        }
        // bucket 找完（还没找到），继续到 overflow bucket 里找
        b = b.overflow(t)
        // overflow bucket 也找完了，说明没有目标 key
        // 返回零值
        if b == nil {
            return unsafe.Pointer(&zeroVal[0])
        }
    }
}
```
函数返回 h[key] 的指针，如果 h 中没有此 key，那就会返回一个 key 相应类型的零值，不会返回 nil。 代码整体比较直接，没什么难懂的地方。跟着上面的注释一步步理解就好了。
这里，说一下定位 key 和 value 的方法以及整个循环的写法。
```go
// key 定位公式
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
// value 定位公式
v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
```
b 是 bmap 的地址，这里 bmap 还是源码里定义的结构体，只包含一个 tophash 数组，经编译器扩充之后的结构体才包含 key，value，overflow 这些字段。dataOffset 是 key 相对于 bmap 起始地址的偏移：
```go
dataOffset = unsafe.Offsetof(struct {
        b bmap
        v int64
    }{}.v)
```
因此 bucket 里 key 的起始地址就是 unsafe.Pointer(b)+dataOffset。第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；而我们又知道，value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。理解了这些，上面 key 和 value 的定位公式就很好理解了。

再说整个大循环的写法，最外层是一个无限循环，通过
```go
b = b.overflow(t)
```
遍历所有的 bucket，这相当于是一个 bucket 链表。

当定位到一个具体的 bucket 时，里层循环就是遍历这个 bucket 里所有的 cell，或者说所有的槽位，也就是 bucketCnt=8 个槽位。整个循环过程：

<img src="images/循环过程.png" alt="循环过程" style="zoom:40%;" />

再说一下 minTopHash，当一个 cell 的 tophash 值小于 minTopHash 时，标志这个 cell 的迁移状态。因为这个状态值是放在 tophash 数组里，为了和正常的哈希值区分开，会给 key 计算出来的哈希值一个增量：minTopHash。这样就能区分正常的 top hash 值和表示状态的哈希值。

下面的这几种状态就表征了 bucket 的情况：
```go
// 空的 cell，也是初始时 bucket 的状态
empty          = 0
// 空的 cell，表示 cell 已经被迁移到新的 bucket
evacuatedEmpty = 1
// key,value 已经搬迁完毕，但是 key 都在新 bucket 前半部分，
// 后面扩容部分会再讲到。
evacuatedX     = 2
// 同上，key 在后半部分
evacuatedY     = 3
// tophash 的最小正常值
minTopHash     = 4
```
源码里判断这个 bucket 是否已经搬迁完毕，用到的函数：
```go
func evacuated(b *bmap) bool {
    h := b.tophash[0]
    return h > empty && h < minTopHash
}
```
只取了 tophash 数组的第一个值，判断它是否在 0-4 之间。对比上面的常量，当 top hash 是 evacuatedEmpty、evacuatedX、evacuatedY 这三个值之一，说明此 bucket 中的 key 全部被搬迁到了新 bucket。

## <span id="jump7">map的赋值过程</span>
通过汇编语言可以看到，向 map 中插入或者修改 key，最终调用的是 mapassign 函数。

实际上插入或修改 key 的语法是一样的，只不过前者操作的 key 在 map 中不存在，而后者操作的 key 存在 map 中。

mapassign 有一个系列的函数，根据 key 类型的不同，编译器会将其优化为相应的“快速函数”。
<img src="./images/key类型插入.png" alt="key类型插入" style="zoom:40%;" />
我们只用研究最一般的赋值函数 mapassign。

整体来看，流程非常得简单：对 key 计算 hash 值，根据 hash 值按照之前的流程，找到要赋值的位置（可能是插入新 key，也可能是更新老 key），对相应位置进行赋值。

源码大体和之前讲的类似，核心还是一个双层循环，外层遍历 bucket 和它的 overflow bucket，内层遍历整个 bucket 的各个 cell。限于篇幅，这部分代码的注释我也不展示了，有兴趣的可以去看，保证理解了这篇文章内容后，能够看懂。

我这里会针对这个过程提几点重要的。

函数首先会检查 map 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。这也说明了 map 对协程是不安全的。

通过前文我们知道扩容是渐进式的，如果 map 处在扩容的过程中，那么当 key 定位到了某个 bucket 后，需要确保这个 bucket 对应的老 bucket 完成了迁移过程。即老 bucket 里的 key 都要迁移到新的 bucket 中来（分裂到 2 个新 bucket），才能在新的 bucket 中进行插入或者更新的操作。

上面说的操作是在函数靠前的位置进行的，只有进行完了这个搬迁操作后，我们才能放心地在新 bucket 里定位 key 要安置的地址，再进行之后的操作。

现在到了定位 key 应该放置的位置了，所谓找准自己的位置很重要。准备两个指针，一个（inserti）指向 key 的 hash 值在 tophash 数组所处的位置，另一个(insertk)指向 cell 的位置（也就是 key 最终放置的地址），当然，对应 value 的位置就很容易定位出来了。这三者实际上都是关联的，在 tophash 数组中的索引位置决定了 key 在整个 bucket 中的位置（共 8 个 key），而 value 的位置需要“跨过” 8 个 key 的长度。

在循环的过程中，inserti 和 insertk 分别指向第一个找到的空闲的 cell。如果之后在 map 没有找到 key 的存在，也就是说原来 map 中没有此 key，这意味着插入新 key。那最终 key 的安置地址就是第一次发现的“空位”（tophash 是 empty）。

如果这个 bucket 的 8 个 key 都已经放置满了，那在跳出循环后，发现 inserti 和 insertk 都是空，这时候需要在 bucket 后面挂上 overflow bucket。当然，也有可能是在 overflow bucket 后面再挂上一个 overflow bucket。这就说明，太多 key hash 到了此 bucket。

在正式安置 key 之前，还要检查 map 的状态，看它是否需要进行扩容。如果满足扩容的条件，就主动触发一次扩容操作。

这之后，整个之前的查找定位 key 的过程，还得再重新走一次。因为扩容之后，key 的分布都发生了变化。

最后，会更新 map 相关的值，如果是插入新 key，map 的元素数量字段 count 值会加 1；在函数之初设置的 hashWriting 写标志出会清零。

另外，有一个重要的点要说一下。前面说的找到 key 的位置，进行赋值操作，实际上并不准确。我们看 mapassign 函数的原型就知道，函数并没有传入 value 值，所以赋值操作是什么时候执行的呢？
```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
```
mapassign 函数返回的指针就是指向的 key 所对应的 value 值位置，有了地址，就很好操作赋值了。
## <span id="jump8">map的遍历过程</span>

本来 map 的遍历过程比较简单：遍历所有的 bucket 以及它后面挂的 overflow bucket，然后挨个遍历 bucket 中的所有 cell。每个 bucket 中包含 8 个 cell，从有 key 的 cell 中取出 key 和 value，这个过程就完成了。

但是，扩容过程不是一个原子的操作，它每次最多只搬运 2 个 bucket，所以如果触发了扩容操作，那么在很长时间里，map 的状态都是处于一个中间态：有些 bucket 已经搬迁到新家，而有些 bucket 还待在老地方。

因此，遍历如果发生在扩容的过程中，就会涉及到遍历新老 bucket 的过程。
先写一个简单的代码样例:
```go
package main
import "fmt"
func main() {
    ageMp := make(map[string]int)
    ageMp["qcrao"] = 18
    for name, age := range ageMp {
        fmt.Println(name, age)
    }
}
```
执行命令：
```go
go tool compile -S main.go
```
得到汇编命令。 关键的几行汇编代码如下：
```go
// ......
0x0124 00292 (test16.go:9)      CALL    runtime.mapiterinit(SB)
// ......
0x01fb 00507 (test16.go:9)      CALL    runtime.mapiternext(SB)
0x0200 00512 (test16.go:9)      MOVQ    ""..autotmp_4+160(SP), AX
0x0208 00520 (test16.go:9)      TESTQ   AX, AX
0x020b 00523 (test16.go:9)      JNE     302
// ......
```
这样，关于 map 迭代，底层的函数调用关系一目了然。先是调用 mapiterinit 函数初始化迭代器，然后循环调用 mapiternext 函数进行 map 迭代。

<img src="./images/底层_遍历_1.png" alt="底层_遍历_1" style="zoom:40%;" width="80" height="308"/>

迭代器的结构体定义：
```go
type hiter struct {
    // key 指针
    key         unsafe.Pointer
    // value 指针
    value       unsafe.Pointer
    // map 类型，包含如 key size 大小等
    t           *maptype
    // map header
    h           *hmap
    // 初始化时指向的 bucket
    buckets     unsafe.Pointer
    // 当前遍历到的 bmap
    bptr        *bmap
    overflow    [2]*[]*bmap
    // 起始遍历的 bucet 编号
    startBucket uintptr
    // 遍历开始时 cell 的编号（每个 bucket 中有 8 个 cell）
    offset      uint8
    // 是否从头遍历了
    wrapped     bool
    // B 的大小
    B           uint8
    // 指示当前 cell 序号
    i           uint8
    // 指向当前的 bucket
    bucket      uintptr
    // 因为扩容，需要检查的 bucket
    checkBucket uintptr
}
```
mapiterinit 就是对 hiter 结构体里的字段进行初始化赋值操作。

前面已经提到过，即使是对一个写死的 map 进行遍历，每次出来的结果也是无序的。下面我们就可以近距离地观察他们的实现了。

```go
// 生成随机数 r
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
    r += uintptr(fastrand()) << 31
}
// 从哪个 bucket 开始遍历
it.startBucket = r & (uintptr(1)<<h.B - 1)
// 从 bucket 的哪个 cell 开始遍历
it.offset = uint8(r >> h.B & (bucketCnt - 1))
```
例如，B = 2，那 uintptr(1)<<h.B - 1 结果就是 3，低 8 位为 0000 0011，将 r 与之相与，就可以得到一个 0~3 的 bucket 序号；bucketCnt - 1 等于 7，低 8 位为 0000 0111，将 r 右移 2 位后，与 7 相与，就可以得到一个 0~7 号的 cell。

于是，在 mapiternext 函数中就会从 it.startBucket 的 it.offset 号的 cell 开始遍历，取出其中的 key 和 value，直到又回到起点 bucket，完成遍历过程。

源码部分比较好看懂，尤其是理解了前面注释的几段代码后，再看这部分代码就没什么压力了。所以，接下来，我将通过图形化的方式讲解整个遍历过程，希望能够清晰易懂。

假设我们有下图所示的一个 map，起始时 B = 1，有两个 bucket，后来触发了扩容（这里不要深究扩容条件，只是一个设定），B 变成 2。并且， 1 号 bucket 中的内容搬迁到了新的 bucket，1 号裂变成 1 号和 3 号；0 号 bucket 暂未搬迁。老的 bucket 挂在在 *oldbuckets 指针上面，新的 bucket 则挂在 *buckets 指针上面。
<img src="./images/底层_遍历_2.png" alt="底层_遍历_2" style="zoom:40%;" />
这时，我们对此 map 进行遍历。假设经过初始化后，startBucket = 3，offset = 2。于是，遍历的起点将是 3 号 bucket 的 2 号 cell，下面这张图就是开始遍历时的状态：
<img src="./images/底层_遍历_3.png" alt="底层_遍历_3" style="zoom:40%;" />
标红的表示起始位置，bucket 遍历顺序为：3 -> 0 -> 1 -> 2。

因为 3 号 bucket 对应老的 1 号 bucket，因此先检查老 1 号 bucket 是否已经被搬迁过。判断方法就是：
```go
func evacuated(b *bmap) bool {
    h := b.tophash[0]
    return h > empty && h < minTopHash
}
```
如果 b.tophash[0] 的值在标志值范围内，即在 (0,4) 区间里，说明已经被搬迁过了。
```go
empty = 0
evacuatedEmpty = 1
evacuatedX = 2
evacuatedY = 3
minTopHash = 4
```
在本例中，老 1 号 bucket 已经被搬迁过了。所以它的 tophash[0] 值在 (0,4) 范围内，因此只用遍历新的 3 号 bucket。

依次遍历 3 号 bucket 的 cell，这时候会找到第一个非空的 key：元素 e。到这里，mapiternext 函数返回，这时我们的遍历结果仅有一个元素：

<img src="./images/底层_遍历_4.png" alt="底层_遍历_4" style="zoom:40%;" />

由于返回的 key 不为空，所以会继续调用 mapiternext 函数。

继续从上次遍历到的地方往后遍历，从新 3 号 overflow bucket 中找到了元素 f 和 元素 g。

遍历结果集也因此壮大：

<img src="./images/底层_遍历_5.png" alt="底层_遍历_5" style="zoom:40%;" />

新 3 号 bucket 遍历完之后，回到了新 0 号 bucket。0 号 bucket 对应老的 0 号 bucket，经检查，老 0 号 bucket 并未搬迁，因此对新 0 号 bucket 的遍历就改为遍历老 0 号 bucket。那是不是把老 0 号 bucket 中的所有 key 都取出来呢？

并没有这么简单，回忆一下，老 0 号 bucket 在搬迁后将裂变成 2 个 bucket：新 0 号、新 2 号。而我们此时正在遍历的只是新 0 号 bucket（注意，遍历都是遍历的 *bucket 指针，也就是所谓的新 buckets）。所以，我们只会取出老 0 号 bucket 中那些在裂变之后，分配到新 0 号 bucket 中的那些 key。

因此，lowbits == 00 的将进入遍历结果集：

<img src="./images/底层_遍历_6.png" alt="底层_遍历_6" style="zoom:40%;" />

和之前的流程一样，继续遍历新 1 号 bucket，发现老 1 号 bucket 已经搬迁，只用遍历新 1 号 bucket 中现有的元素就可以了。结果集变成：

<img src="./images/底层_遍历_7.png" alt="底层_遍历_7" style="zoom:40%;" />

继续遍历新 2 号 bucket，它来自老 0 号 bucket，因此需要在老 0 号 bucket 中那些会裂变到新 2 号 bucket 中的 key，也就是 lowbit == 10 的那些 key。

这样，遍历结果集变成：

<img src="./images/底层_遍历_8.png" alt="底层_遍历_8" style="zoom:40%;" />

最后，继续遍历到新 3 号 bucket 时，发现所有的 bucket 都已经遍历完毕，整个迭代过程执行完毕。

顺便说一下，如果碰到 key 是 math.NaN() 这种的，处理方式类似。核心还是要看它被分裂后具体落入哪个 bucket。只不过只用看它 top hash 的最低位。如果 top hash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。据此决定是否取出 key，放到遍历结果集里。

map 遍历的核心在于理解 2 倍扩容时，老 bucket 会分裂到 2 个新 bucket 中去。而遍历操作，会按照新 bucket 的序号顺序进行，碰到老 bucket 未搬迁的情况时，要在老 bucket 中找到将来要搬迁到新 bucket 来的 key。
## <span id="jump9">map的删除过程</span>
## <span id="jump10">map的扩容过程</span>


