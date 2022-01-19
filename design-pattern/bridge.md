# bridge

桥接模式，类图很简单，但是很多资料都在说是可以多个维度独立的变化，不理解的时候很懵，但是理解后发现概括的确实很好，对于不理解的人来说感觉有点太过于抽象了，那么可以换一个角度来说，从类图来看，就是将一系列紧密相关的类拆分为抽象和实现两个独立的层次结构（当然了，这里的“实现”是抽象的，这里的“抽象”是更高级的抽象），个人认为“实现”这个词在这里容易引起歧义。更直白的说，是将多个不同的维度的角色进行抽象，然后利用更高级的抽象将它们关联起来，并在这些的基础上有一些升级。

其实看了上面的文字可能还不是太直观，先看一下使用桥接模式前后的差距吧。

![image](https://github.com/Saner-Lee/pictures/raw/master/bridge_before.png)

![image](https://github.com/Saner-Lee/pictures/raw/master/bridge_after.png)

图片只显示了两个维度（可以理解为对象的两个属性，一个属性变量代表一个维度，只不过其中一个属性是接口），其实可以是多个维度，那么只需要`Shape`包含其它维度的接口即可。

```go
type Role int

const (
	VIP = iota + 1
	Normal
)

type CashSystem interface {
	Deducation(Role, float64)
}

type XxxCash struct {
	pay Pay
}

func NewXxxCash(pay Pay) CashSystem {
	return &XxxCash{pay: pay}
}

func (cashier XxxCash) Deducation(r Role, money float64) {
	if r == VIP {
		cashier.pay.Cost(0.9 * money)
		return
	}
	cashier.pay.Cost(money)
}

type Pay interface {
	Cost(float64)
}

type AliPay struct{}

func (pay *AliPay) Cost(money float64) {
	fmt.Printf("支付宝消费 : %.2fRMB\n", money)
}

type TencentPay struct{}

func (pay *TencentPay) Cost(money float64) {
	fmt.Printf("微信消费 : %.2fRMB\n", money)
}

func main() {
	x := NewXxxCash(new(AliPay))
	y := NewXxxCash(new(TencentPay))

	x.Deducation(Normal, 100)
	x.Deducation(VIP, 100)
	y.Deducation(Normal, 100)
	y.Deducation(VIP, 100)
}
/* ——————————下面是输出---------- */
// 支付宝消费 : 100.00RMB
// 支付宝消费 : 90.00RMB
// 微信消费 : 100.00RMB
// 微信消费 : 90.00RMB
```

上面这个例子基本删和之前给的图是完全符合的，其中的类换了一下，只不过在`go`中并没有抽象类，所以将包含关系的位置变动了一下。

