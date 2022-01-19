# strategy

策略模式是最简单的一个设计模式了，如果了解多态，就能掌握这种设计模式，应该作为入门设计模式的第一个设计模式。

![image](https://github.com/Saner-Lee/pictures/raw/master/strategy_pattern_uml.png)

只是简单的使用了一个接口，将算法和实现解耦。

```go
type Client struct {
	name    string
	payment Payment
}

func NewClient(name string, payment Payment) *Client {
	return &Client{
		name:    name,
		payment: payment,
	}
}

func (c *Client) SetPayment(payment Payment) {
	if payment == nil {
		return
	}
	c.payment = payment
}

func (c *Client) Cost(money float64) {
	fmt.Printf("%s cost %.2f RMB\n", c.name, money)
	c.payment.Pay(money)
}

type Payment interface {
	Pay(float64)
}

type Tencent struct{}

func (pm *Tencent) Pay(money float64) {
	fmt.Printf("tencent earned %.2f RMB\n", money)
}

type Alibaba struct{}

func (pm *Alibaba) Pay(money float64) {
	fmt.Printf("Ali earned %.2f RMB\n", money)
}

func main() {
	lyh := NewClient("liyanhong", new(Alibaba))
	lyh.Cost(12345.67)

	lyh.SetPayment(new(Tencent))
	lyh.Cost(76543.21)
}
// output :
// liyanhong cost 12345.67 RMB
// Ali earned 12345.67 RMB
// liyanhong cost 76543.21 RMB
// tencent earned 76543.21 RMB
```



