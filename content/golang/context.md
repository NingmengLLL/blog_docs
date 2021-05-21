```json
{
  "date": "2021.05.21 15:00",
  "tags": ["go","context",""],
  "description":"Go 1.7 标准库引入 context，中文译作“上下文”，准确说它是 goroutine 的上下文，包含 goroutine 的运行状态、环境、现场等信息。context 主要用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v 等。随着 context 包的引入，标准库中很多接口因此加上了 context 参数，例如 database/sql 包。context 几乎成为了并发控制和超时控制的标准做法。"
}
```
[context源码](#jump1)

&emsp; &emsp; [接口](#jump1_1)

&emsp; &emsp; &emsp; &emsp; [Context](#jump1_1_1)

&emsp; &emsp; &emsp; &emsp; [canceler](#jump1_1_2)

&emsp; &emsp; [结构体](#jump1_2)

&emsp; &emsp; &emsp; &emsp; [emptyCtx](#jump1_2_1)

&emsp; &emsp; &emsp; &emsp; [cancelCtx](#jump1_2_2)

&emsp; &emsp; &emsp; &emsp; [timerCtx](#jump1_2_3)

[context的应用](#jump2)

&emsp; &emsp; [传递共享的数据](#jump2_1)

&emsp; &emsp; [取消goroutine](#jump2_2)

&emsp; &emsp; [防止goroutine泄漏](#jump2_3)

[context.Value的查找过程](#jump3)

# <span id="jump1">context源码</span>

## <span id="jump1_1">接口</span>

### <span id="jump1_1_1">Context</span>

### <span id="jump1_1_2">canceler</span>

## <span id="jump1_2">结构体</span>

### <span id="jump1_2_1">emptyCtx</span>

### <span id="jump1_2_2">cancelCtx</span>

### <span id="jump1_2_3">timerCtx</span>

# <span id="jump2">context的应用</span>

## <span id="jump2_1">传递共享的数据</span>

## <span id="jump2_2">取消goroutine</span>

## <span id="jump2_3">防止goroutine泄漏</span>

# <span id="jump3">context.Value的查找过程</span>
