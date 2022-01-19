# observer

观察者模式主要用来解决一对多的依赖关系，当一个对象状态更新时，所有依赖它的对象都会收到通知并自动更新。



## 设计

![image](https://github.com/Saner-Lee/pictures/raw/master/1.png)

## 本质

本质是信息的流式传递。核心操作是观察者的注册，核心思想是只通知注册的观察者。使用比较高效的“注册-推送”机制替代了相对低效的“关注-拉取”机制。

## 实现

### 被观察者

```go
type Subject interface {
	Attach(Observer)
	Detach(Observer)
	Notify()
}

type Idol struct {
	fans    map[Observer]struct{}
	address string
}

func NewIdol() *Idol {
	return &Idol{address: "还未确定"}
}

func (i *Idol) Attach(o Observer) {
	if i.fans == nil {
		i.fans = make(map[Observer]struct{})
	}

	if _, exist := i.fans[o]; exist {
		return
	}

	i.fans[o] = struct{}{}
}

func (i *Idol) Detach(o Observer) {
	if i.fans == nil {
		return
	}

	if _, exist := i.fans[o]; exist {
		delete(i.fans, o)
	}
}

func (i *Idol) Notify() {
	if i.fans == nil {
		return
	}

	for k, _ := range i.fans {
		k.Update(i.address)
	}
}

func (i *Idol) SetAddress(addr string) {
	i.address = addr
}
```



### 观察者

```go
type Observer interface {
	Update(addr string)
}

type Fan struct {
	name string
}

func NewFan(name string) *Fan {
	return &Fan{name}
}

func (f *Fan) Update(addr string) {
	log.Println(fmt.Sprintf("%s，你偶像的下一站演唱会地点：%s", f.name, addr))
}
```



### 实例



```go
func TestObserver(t *testing.T) {

	fansA, fansB := NewFan("ltx"), NewFan("xzy")
	idol := NewIdol()

	idol.Attach(fansA)
	idol.SetAddress("北京")
	idol.Notify()

	idol.Attach(fansB)
	idol.Detach(fansA)
	idol.SetAddress("上海")
	idol.Notify()
}
```



### 输出

```go
2019/05/05 19:33:49 ltx，你偶像的下一站演唱会地点：北京
2019/05/05 19:33:49 xzy，你偶像的下一站演唱会地点：上海
```



