```json
{
  "date": "2021.05.19 13:23",
  "tags": ["golang","map","float"],
  "description":"从语法上看，是可以的。Go 语言中只要是可比较的类型都可以作为 key。除开 slice，map，functions 这几种类型，其他类型都是 OK 的。"
}
```

看个例子
```go
func main() {
    m := make(map[float64]int)
    m[1.4] = 1
    m[2.4] = 2
    m[math.NaN()] = 3
    m[math.NaN()] = 3
    for k, v := range m {
    fmt.Printf("[%v, %d] ", k, v)
    }
    fmt.Printf("\nk: %v, v: %d\n", math.NaN(), m[math.NaN()])
    fmt.Printf("k: %v, v: %d\n", 2.400000000001, m[2.400000000001])
    fmt.Printf("k: %v, v: %d\n", 2.4000000000000000000000001, m[2.4000000000000000000000001])
    fmt.Println(math.NaN() == math.NaN())
}
```
程序输出
```go
[2.4, 2] [NaN, 3] [NaN, 3] [1.4, 1]
k: NaN, v: 0
k: 2.400000000001, v: 0
k: 2.4, v: 2
false
```
当用 float64 作为 key 的时候，先要将其转成 unit64 类型，再插入 key 中。 具体是通过 Float64frombits 函数完成：
```go
// Float64frombits returns the floating point number corresponding
// the IEEE 754 binary representation b.
func Float64frombits(b uint64) float64 { return *(*float64)(unsafe.Pointer(&b)) }
```
我们再来输出点东西：
```go
func main() {
    m := make(map[float64]int)
    m[2.4] = 2
    fmt.Println(math.Float64bits(2.4))
    fmt.Println(math.Float64bits(2.400000000001))
    fmt.Println(math.Float64bits(2.4000000000000000000000001))
}

4612586738352862003
4612586738352864255
4612586738352862003
```
2.4 和 2.4000000000000000000000001 经过 math.Float64bits() 函数转换后的结果是一样的。自然，二者在 map 看来，就是同一个 key 了。

再来看一下 NAN（not a number）：
```go
// NaN returns an IEEE 754 ``not-a-number'' value.
func NaN() float64 { return Float64frombits(uvnan) }
```
uvan 的定义为：
```go
uvnan    = 0x7FF8000000000001
```
NAN() 直接调用 Float64frombits，传入写死的 const 型变量 0x7FF8000000000001，得到 NAN 型值。既然，NAN 是从一个常量解析得来的，为什么插入 map 时，会被认为是不同的 key？

这是由类型的哈希函数决定的，例如，对于 64 位的浮点数，它的哈希函数如下：
```go
func f64hash(p unsafe.Pointer, h uintptr) uintptr {
    f := *(*float64)(p)
    switch {
    case f == 0:
        return c1 * (c0 ^ h) // +0, -0
    case f != f:
        return c1 * (c0 ^ h ^ uintptr(fastrand())) // any kind of NaN
    default:
        return memhash(p, h, 8)
    }
}
```
第二个 case，f != f 就是针对 NAN，这里会再加一个随机数。 这样，所有的谜题都解开了。 由于 NAN 的特性：
```go
NAN != NAN
hash(NAN) != hash(NAN)
```
因此向 map 中查找的 key 为 NAN 时，什么也查不到；如果向其中增加了 4 次 NAN，遍历会得到 4 个 NAN。

最后说结论：float 型可以作为 key，但是由于精度的问题，会导致一些诡异的问题，慎用之。