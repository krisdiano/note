# cron定时任务

基于cron表达式的定时任务框架。`https://godoc.org/github.com/robfig/cron`

## 接口

有两个抽象出来的接口

- Job
- Schedule

### Job

```go
type Job interface {
	Run()
}
```

### Schedule

```go
type Schedule interface {
	// Return the next activation time, later than the given time.
	// Next is invoked initially, and then each time the job is run.
	Next(time.Time) time.Time
}
```

## 结构

有两个主要的结构：

- Entry
- Cron

### Entry

Entry是定时任务中的一个节点，结构组成如下：

```go
type Entry struct {
	Schedule Schedule
	Next time.Time
	Prev time.Time
	Job Job
}
```

### Cron

Cron是一个定时任务管理的实例，结构组成如下：

```go
type Cron struct {
	entries  []*Entry
	stop     chan struct{}
	add      chan *Entry
	snapshot chan []*Entry
	running  bool
	ErrorLog Logger
	location *time.Location
}
```

## 代码分析

### 构造定时器实例

构造的方法实现比较简单，没有什么好说的。

```go
func New() *Cron {
	return NewWithLocation(time.Now().Location())
}

func NewWithLocation(location *time.Location) *Cron {
	return &Cron{
		entries:  nil,
		add:      make(chan *Entry),
		stop:     make(chan struct{}),
		snapshot: make(chan []*Entry),
		running:  false,
		ErrorLog: nil,
		location: location,
	}
}
```

### 添加任务

对外提供了两种添加任务的方式：

- AddJob
- AddFunc

后者相较于前者使用更加方便，不过本质确实对前者的一个封装。

#### AddJob

根据spec这个cron表达式解析出一个Schedule实例，然后将Schedule实例和到期执行的Job添加到一个新的Entry中。

```go
func (c *Cron) AddJob(spec string, cmd Job) error {
	schedule, err := Parse(spec)
	if err != nil {
		return err
	}
	c.Schedule(schedule, cmd)
	return nil
}

func (c *Cron) Schedule(schedule Schedule, cmd Job) {
	entry := &Entry{
		Schedule: schedule,
		Job:      cmd,
	}
    	// 没有run就直接添加到它的entries切片中，否则利用通道传递entry
	if !c.running {
		c.entries = append(c.entries, entry)
		return
	}

	c.add <- entry
}
```

#### AddFunc

使用对函数类型的在定义对外提供了一种比较友好的方式，这样的话使用者无需定义结构实现Job接口。

```go
type FuncJob func()

func (f FuncJob) Run() { f() }

func (c *Cron) AddFunc(spec string, cmd func()) error {
    	// 看似没有什么知识，其实这里的强转设计到了go的Assignability和underlying types
	return c.AddJob(spec, FuncJob(cmd))
}
```

### 监控

监控函数其实是包内的一个私有函数，负责cron的运行，监控cron的行为。

```go
func (c *Cron) run() {
	// 根据当前时间计算出每个entry的到期时间
	now := c.now()
	for _, entry := range c.entries {
		entry.Next = entry.Schedule.Next(now)
	}

	for {
		// 将entries中的所有节点按照到期时间排序
		sort.Sort(byTime(c.entries))

		var timer *time.Timer
		if len(c.entries) == 0 || c.entries[0].Next.IsZero() {
			// 如果没有节点就休眠
            		// IsZero可能是因为Schedule对外开放的方法
            		// 使用者如果没有正确实现Schedule接口可能导致任务一直执行
			timer = time.NewTimer(100000 * time.Hour)
		} else {
			timer = time.NewTimer(c.entries[0].Next.Sub(now))
		}

		for {
			select {
			case now = <-timer.C:
				now = now.In(c.location)
				// 此时的entries是按照到期时间排序的
				for _, e := range c.entries {
                    			// 按照顺序处理直到遇到第一个未到期的节点
					if e.Next.After(now) || e.Next.IsZero() {
						break
					}
                    			// 处理到期的节点
					go c.runWithRecovery(e.Job)
					e.Prev = e.Next
					e.Next = e.Schedule.Next(now)
				}

            		// 在select过程中如果执行了添加节点的操作
            		// 关闭定时器，计算到期时间并且添加到entries切片中
            		// 这个信号会导致退出此次select然后回到最外层for循环中重新排序，可能导致任务延期执行
			case newEntry := <-c.add:
				timer.Stop()
				now = c.now()
				newEntry.Next = newEntry.Schedule.Next(now)
				c.entries = append(c.entries, newEntry)

			case <-c.snapshot:
				c.snapshot <- c.entrySnapshot()
				continue

            		// 收到stop信号关闭定时器退出
			case <-c.stop:
				timer.Stop()
				return
			}

			break
		}
	}
}
```

### 其它

这里包含的函数一般就是启动，停止，获取节点，它们都和监控函数紧密相关。

#### Run

```go
func (c *Cron) Run() {
	if c.running {
		return
	}
	c.running = true
	c.run()
}
```

#### Stop

```go
func (c *Cron) Stop() {
	if !c.running {
		return
	}
	c.stop <- struct{}{}
	c.running = false
}
```

#### Entries

使用者使用这个函数其实是想获得其中的节点的详细信息，但是为了使用者获得节点信息后改动其中的值而影响了监控函数的正确的执行，所以其实返回的是一个深拷贝。

```go
func (c *Cron) Entries() []*Entry {
	if c.running {
		c.snapshot <- nil
        	// 上面通过sanpshot通道传递给监控函数一个信号
        	// 监控函数接收到信号返回了对切片的一个深拷贝
		x := <-c.snapshot
		return x
	}
	return c.entrySnapshot()
}

// 深拷贝，这样使用者改变了Next的值也不会影响定时任务正确的执行
func (c *Cron) entrySnapshot() []*Entry {
	entries := []*Entry{}
	for _, e := range c.entries {
		entries = append(entries, &Entry{
			Schedule: e.Schedule,
			Next:     e.Next,
			Prev:     e.Prev,
			Job:      e.Job,
		})
	}
	return entries
}
```

## 总结

这里并没有涉及怎么解析表达式，目的是为了理解怎么 涉及定时任务的处理。

阅读后有一些理解，分享一下。cron的每个实例都维护了一个线性表，线性表中的节点按照到期时间排序，且每个实例Run之后都会起一个ioLoop的goroutine，用于监测其它goroutine对实例的执行的动作，一旦这个动作可能打破当前线性表维护的顺序，那么就会重新排序然后继续监测。

在很多源码中都能看到类似的操作，比如nsq也是这样使用channel。应该掌握channel的这种使用方式。

## 疑问

为什么使用了最简单的排序，而非小根堆或者定时轮的结构，难道因为使用的场景定时任务数量不会太大，排序的性能完全可以支持？

















