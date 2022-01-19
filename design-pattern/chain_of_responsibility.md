# chain of responsibility

主要分为下述两种：

- 标准责任链
- 变形责任链

其核心思想都是处理链，不同点在于请求如何在链上传递。

## 标准责任链

链上的所有节点都有可能处理请求，但是至多有一个节点最终处理请求。适用范围比较小，不深入了解。

## 变形责任链

链上的所有节点都要对请求进行处理。相比于标准形式更通用。也可以细分为两类：

- 单向处理请求
- 请求-响应

### 单向处理请求

#### 接口

处理链上的每个节点都可以处理请求，那么必然有一个相同的接口可以抽象出来。

```go
type Filter interface {
	DoFilter(string) string
}
```

#### 处理链

```go
type FilterChain struct {
	chain *list.List	//自组合需要的过滤器
}

func NewFilterChain() *FilterChain {
	return &FilterChain{list.New()}
}

// 添加一个Filter实例
// 为了实现链式调用，返回自身
func (f *FilterChain) Add(filter Filter) *FilterChain {
	f.chain.PushBack(filter)
	return f
}

// FilterChain也实现了Filter接口，这样一条链可以直接添加另外一条链
func (f *FilterChain) DoFilter(data string) string {
	for ele := f.chain.Front(); ele != nil; ele = ele.Next() {
		filter := ele.Value.(Filter)
		data = filter.DoFilter(data)
	}

	return data
}
```

#### 测试

下面是为了测试准备的一些Filter实例

```go
type HTMLFilter struct{}

// 将尖括号替换为方括号
func (h *HTMLFilter) DoFilter(data string) string {
	return strings.Replace(strings.Replace(data, "<", "[", -1), ">", "]", -1)
}

type SensitiveFilter struct{}

// 将被就业替换为就业
func (s *SensitiveFilter) DoFilter(data string) string {
	return strings.Replace(data, "被就业", "就业", -1)
}

// 将/笑脸替换为Smile
type SmileFilter struct{}

func (s *SmileFilter) DoFilter(data string) string {
	return strings.Replace(data, "/笑脸", "Smile", -1)
}
```

下面是测试用例

```go
func TestSimple(t *testing.T) {

	input := "<scrpit>xxxxx</scrpit>   被就业  /笑脸"

	fc := NewFilterChain()
	fc.Add(new(HTMLFilter)).Add(new(SensitiveFilter))
	t.Log(fc.DoFilter(input))

	fcc := NewFilterChain()
	fcc.Add(new(SmileFilter)).Add(fc)
	t.Log(fcc.DoFilter(input))
}

simple_test.go:11: [scrpit]xxxxx[/scrpit]   就业  /笑脸
simple_test.go:15: [scrpit]xxxxx[/scrpit]   就业  Smile
```

### 请求-响应

#### 接口

处理链上的每个节点都要处理请求和响应，抽象出以下接口。

```go
type Request struct {
	requ string
}

type Response struct {
	resp string
}

// chain主要是为了可以逆序处理响应
type Filter interface {
	DoFilter(request *Request, response *Response, chain *FilterChain)
}
```

#### 处理链

```go
type FilterChain struct {
	over  bool
	ptr   *list.Element
	chain *list.List
}

func NewFilterChain() *FilterChain {
	return &FilterChain{chain: list.New()}
}

func (f *FilterChain) Add(filter Filter) *FilterChain {
	f.chain.PushBack(filter)
	return f
}

// 如果忘记为什么可以逆序处理响应，可以在FilterChain的DoFilter处DEBUG一下。
func (f *FilterChain) DoFilter(request *Request, response *Response, chain *FilterChain) {
	if f.over {
		f.over = false
		return
	}

	if f.ptr == nil {
		f.ptr = f.chain.Front()
	}

	filter := f.ptr.Value.(Filter)
	if f.ptr = f.ptr.Next(); f.ptr == nil {
		f.over = true
	}
	filter.DoFilter(request, response, chain)
}
```

#### 测试

改进的Filter实例

```go
type HTMLFilter struct{}

func (h *HTMLFilter) DoFilter(request *Request, response *Response, chain *FilterChain) {
	request.requ = strings.Replace(strings.Replace(request.requ, "<", "[", -1), ">", "]", -1)
	chain.DoFilter(request, response, chain)
	response.resp += "   HTML   "
}

type SensitiveFilter struct{}

func (s *SensitiveFilter) DoFilter(request *Request, response *Response, chain *FilterChain) {
	request.requ = strings.Replace(request.requ, "被就业", "就业", -1)
	chain.DoFilter(request, response, chain)
	response.resp += "   Sensitive   "
}

type SmileFilter struct{}

func (s *SmileFilter) DoFilter(request *Request, response *Response, chain *FilterChain) {
	request.requ = strings.Replace(request.requ, "笑脸", "smile", -1)
	chain.DoFilter(request, response, chain)
	response.resp += "   Smile   "
}
```

下面是测试程序

```go
func TestReqResp(t *testing.T) {

	input := "<scrpit>xxxxx</scrpit>   被就业  笑脸"
	requ := new(Request)
	resp := new(Response)
	requ.requ = input

	fc := NewFilterChain()
	fc.Add(new(HTMLFilter)).Add(new(SmileFilter))
	fc.DoFilter(requ, resp, fc)
	t.Log(requ.requ)
	t.Log(resp.resp)
}

double_test.go:15: [scrpit]xxxxx[/scrpit]   被就业  smile
double_test.go:16:    Smile      HTML   
```

## 总结

根据不同的需求，抽象出合适的接口。将可以处理请求的实例按需组成一个处理链，根据不同的场景确定数据如何在链上传递。