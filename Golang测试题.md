# GOLANG 的有趣测试

## 用途

本系列测试题旨在熟悉Golang语法特点，整理原则是找常用的，有意义的，相对其他语言为特色的点，不记录生僻的或者过于基础的技术点。

这些题目可以用作Go程序员面试考点。

这些题目适合对Golang已有一定熟悉的程序员查阅，以了解Golang部分特点。

所有题目均在Golang 1.10.0(Windows/Linux)上测试验证过，题目答案和个人分析会在最后写出，建议先思考再对比结果查阅。

题目是markdown格式，推荐使用本类格式的文本编辑器查阅。

## 题目

### Q1：defer用法

下面代码会输出什么

```
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {

	defer func() {
		fmt.Println("打印前")
	}()

	defer func() {
		fmt.Println("打印中")
	}()

	defer func() {
		fmt.Println("打印后")
	}()

	panic("触发异常")
}
```

**Q1答案**

<details>
<summary>点我显示</summary>

```
panic: 触发异常
打印后
打印中
打印前
```

或

```
打印后
panic: 触发异常
打印中
打印前
```

或

```
打印后
打印中
panic: 触发异常
打印前
```

或

```
打印后
打印中
打印前
panic: 触发异常
```
</details>

**Q1说明**

<details>
<summary>点我显示</summary>

```
- defer 函数会延迟执行，保证在return之前被执行。
- 多个defer函数，会保证 LIFO 顺序被执行。
- panic函数调用执行其实比三个 defer 早，但panic调用后会开启一个新协程进行错误堆栈输出，所以会导致“触发异常”这一行输出顺序无法确定。
```
</details>


--------


### Q2：panic, defer, recover 错误处理三件套

下面代码会输出什么

```
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() {
		if err := recover();err != nil {
			fmt.Println(err)
		}
		fmt.Println("打印前")
	}()

	defer func() {
		if err := recover();err != nil {
			fmt.Println(err)
		}
		fmt.Println("打印中")
	}()

	defer func() { // 必须要先声明defer，否则recover()不能捕获到panic异常
		if err := recover();err != nil {
			fmt.Println(err)
		}
		fmt.Println("打印后")
	}()

	panic("触发异常")
}
```

**Q2答案**

<details>
<summary>点我显示</summary>

```
触发异常
打印后
打印中
打印前
```
</details>

**Q2说明**

<details>
<summary>点我显示</summary>

```
- Go中抛出panic异常，会被defer中的recover()捕获，并且整个程序会正常执行。
- 被抛出的panic异常，仅会被最近的recover()捕获。
```
</details>

#### Q3：值传递

下面代码会输出什么

```
package main
import (
	"fmt"
)
type student struct {
	Name string
	Age  int
}
func pase_student() map[string]*student {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}
	return m
}
func main() {
	students := pase_student()
	for k, v := range students {
		fmt.Printf("key=%s,value=%v \n", k, v)
	}
}
```

**Q3答案**

<details>
<summary>点我显示</summary>

```
key=zhou,value=&{wang 22} 
key=li,value=&{wang 22} 
key=wang,value=&{wang 22} 
```
</details>

**Q3说明**

<details>
<summary>点我显示</summary>

```
- 在for循环遍历赋值时，每次stu拷贝都是值拷贝，所以 m[stu.Name] = &stu 实际指向的是同一个地址。如果不能理解，可以修改代码为：
	for _, stu := range stus {
	    fmt.Printf("%v\t%p\n",stu,&stu)
		m[stu.Name] = &stu
	}
- 所以，修复该BUG，可以修改代码如下即可：
	for i, _ := range stus {
		stu:=stus[i]
		m[stu.Name] = &stu
	}
```
</details>

#### Q4：变量作用域

下面代码会输出什么

```
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(1)

	count := 10
	wg := sync.WaitGroup{}
	wg.Add(count * 2)
	for i := 0; i < count; i++ {
		go func() {
			fmt.Printf("[%d]", i)
			wg.Done()
		}()
	}
	for i := 0; i < count; i++ {
		go func(i int) {
			fmt.Printf("-%d-", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

**Q4答案**

<details>
<summary>点我显示</summary>

```
-9-[10][10][10][10][10][10][10][10][10][10]-0--1--2--3--4--5--6--7--8-
```
</details>

**Q4说明**

<details>
<summary>点我显示</summary>

```
- 因为设置了runtime CPU数为1，所以整个程序基本可以认为是接近单线程执行的。
- 第一部分的循环，输出的都是[10]，原因是其中print的i不是传参进来的，所以我们可以理解这个循环中执行的是：首先，先10次循环创建go协程，同时i叠加到10；i的遍历循环是非常快的，很快就会到10；然后go协程开始执行，分别print了1次i（此时i已经等于10）
- 第二部分的循环，其中print的i是传参进来的，所以i进行了值拷贝，所以go func()中会输出 0,1,2 ... 9
- 至于为什么第二次循环的 ‘9’ 会首先输出（这是必现），……抱歉，我也不知道。求大神分享讲解。
```
</details>

#### Q5：继承与组合

下面代码会输出什么

```
package main

import "fmt"

type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
	t.ShowB()
}
```

**Q5答案**

<details>
<summary>点我显示</summary>

```
showA
showB
teacher showB
```
</details>

**Q5说明**

<details>
<summary>点我显示</summary>

```
- Go中，没有继承，只有组合。
- 所以上面代码中的 t.ShowA() 其实是 t.People.Show() 的语法糖，两者完全等价。
- 那么下一个值得思考的问题是，如果我们又定义一个结构
	type Test struct{}
	func (t* Test) ShowA(){ fmt.Println("test showA") }
  然后将Test也组合到teacher里
	type Teacher struct{
		People
		Test
	}
  此时，我们调用 t.ShowA() 会输出什么呢？会是 t.People.ShowA() 还是 t.Test.Show()呢？或是有其他情况？留给大家思考。
```
</details>


#### Q6：Select并发通信控制

下面代码会输出什么

```
package main

import (
	"runtime"
	"fmt"
)

func main() {
	runtime.GOMAXPROCS(1)
	
	int_chan := make(chan int, 1)
	string_chan := make(chan string, 1)
	int_chan <- 1
	string_chan <- "hello"
	select {
	case value := <-int_chan:
		fmt.Println(value)
	case value := <-string_chan:
		panic(value)
	}
}
```

**Q6答案**

<details>
<summary>点我显示</summary>

```
1
```
或
```
panic: hello
```
</details>

**Q6说明**

<details>
<summary>点我显示</summary>

```
- select 会随机执行一个可运行的 case，并没有规则和顺序。
- 如果没有default，且没有case可执行，select将会被阻塞。
- 任意一个case可以被执行的话，这个case将会被执行，且其他所有case都将被直接忽略。
```
</details>

#### Q7：defer的参数

下面代码会输出什么

```
package main

import "fmt"

func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, " a=", a, " b=", b, " ret=",ret)
	return ret
}

func main() {
	a := 1
	b := 2
	defer calc("1", a, calc("10", a, b))
	a = 0
	defer calc("2", a, calc("20", a, b))
	b = 1
}
```

**Q7答案**

<details>
<summary>点我显示</summary>

```
10  a= 1  b= 2  ret= 3
20  a= 0  b= 2  ret= 2
2  a= 0  b= 2  ret= 2
1  a= 1  b= 3  ret= 4
```
</details>

**Q7说明**

<details>
<summary>点我显示</summary>

```
- defer 是在return之前执行的，是 FILO 顺序执行的
- defer 函数参数不受 defer 影响
- defer 函数参数是值传递
```
</details>

#### Q8：数组

下面代码会输出什么

```
package main

import "fmt"

func main() {
	s := make([]int, 5)

	fmt.Printf("%p %p %p\n", &s, s, &s[0])
	fmt.Println(s)

	s = append(s, 1)

	fmt.Printf("%p %p %p\n", &s, s, &s[0])
	fmt.Println(s)
}
```

**Q8答案**

<details>
<summary>点我显示</summary>

```
0xc0420503e0 0xc042074030 0xc042074030
[0 0 0 0 0]
0xc0420503e0 0xc042086000 0xc042086000
[0 0 0 0 0 1]
```
</details>

**Q8说明**

<details>
<summary>点我显示</summary>

```
- 数组对象 s 其实是一个指针。
- append 会导致 s 的指向的地址发生更变，所以append并不是简单的数组后面添加数据，而是重新拷贝。
```
</details>

#### Q9：锁

下面代码会输出什么

```
package main

import (
	"fmt"
	"sync"
)

type UserAges struct {
	ages map[string]int
	sync.Mutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}

func main() {
	count := 1000
	gw := sync.WaitGroup{}
	gw.Add(count * 3)
	u := UserAges{ages: map[string]int{}}
	add := func(i int) {
		u.Add(fmt.Sprintf("user_%d", i), i)
		gw.Done()
	}

	for i := 0; i < count; i++ {
		go add(i)
		go add(i)
	}

	for i := 0; i < count; i++ {
		go func(i int) {
			defer gw.Done()
			u.Get(fmt.Sprintf("user_%d", i))
		}(i)
	}
	gw.Wait()
	fmt.Println("Done")
}
```

**Q9答案**

<details>
<summary>点我显示</summary>

```
Done
```
或
```
fatal error: concurrent map read and map write
```
</details>

**Q9说明**

<details>
<summary>点我显示</summary>

```
- 该代码有一定概率成功，也有一定概率会 panic 报错，报错处会是在 Get() 函数内。count数越大，则 panic 报错概率越高。
- 我们使用了Sync.Mutex做锁，但该锁仅仅锁了写，却没有锁读操作。Go中的map是并发读写都不安全的。所以当多个协程访问同一地址时，会出现读写资源竞争，所以会报错：concurrent map read and map write。
- 我们要解决这个问题的话，在Go 1.9 之前版本可以使用 sync.RWMutex 读写锁，它允许一写多读。所以我们只需将上述代码中的sync.Mutex修改为 RWMutex ，然后在读取处添加读锁即可，代码如下：
	func (ua *UserAges) Get(name string) int {
		ua.RLock()
		defer ua.RUnlock()
		if age, ok := ua.ages[name]; ok {
			return age
		}
		return -1
	}
- 另外，如果在Go 1.9之后，则直接使用 sync.map 即可。它和原生的map使用起来相同，但是是协程读写安全的。
```
</details>

#### Q10：Channel

下面代码会输出什么

```
package main

import (
	"fmt"
	"sync"
	"time"
)

type ThreadSafeSet struct {
	sync.RWMutex
	s []int
}

func (set *ThreadSafeSet) Iter() <-chan interface{} {
	ch := make(chan interface{})
	go func() {
		set.RLock()

		for elem, _ := range set.s {
			ch <- elem
			fmt.Print("put into channel:", elem, "\n")
		}

		close(ch)
		set.RUnlock()

	}()
	return ch
}

func main() {
	set := ThreadSafeSet{}
	set.s = make([]int, 100)
	ch := set.Iter()
	_ = ch
	time.Sleep(2 * time.Second)
	fmt.Print("Done\n")
}
```

**Q10答案**

<details>
<summary>点我显示</summary>

```
Done
```
</details>

**Q10说明**

<details>
<summary>点我显示</summary>

```
- 代码不会执行 put into channel，因为创建 channel 时，我们没有使用 ch := make(chan interface{},1)，我们创建的是一个 ch := make(chan interface{}) 空缓冲区的channel，这样的后果是：只要没有接受者(消费者)，插入时会被持续Block。最终，直到主线程因为2秒等待结束，直接退出，销毁协程，协程中的插入行为也无法进行。
- 修改案只要扩大channel缓冲区足够即可，例如 ch := make(chan interface{},100)
- 修改案二，则是增加一个接收方（消费者）。将 _ = ch 该行改为下述代码即可：
	closed := false
	for {
		select {
		case v, ok := <-ch:
			if ok {
				fmt.Print("read:", v, ",")
			} else {
				closed = true
			}
		}
		if closed {
			fmt.Print("closed")
			break
		}
	}
```
</details>


#### Q11：结构体方法和指针方法

下面代码会输出什么

```
package main
import (
	"fmt"
)
type People interface {
	Speak(string) string
}

type YoungMan struct{}
func (stu *YoungMan) Speak(think string) (talk string) {
	if think == "bitch" {
		talk = "You are a bitch"
	} else {
		talk = "hi"
	}
	return
}

type OldMan struct{}
func (stu OldMan) Speak(think string) (talk string) {
	if think == "bitch" {
		talk = "You are a beautiful woman"
	} else {
		talk = "hi"
	}
	return
}

func main() {
	var young1 People = &YoungMan{}
	think := "bitch"
	fmt.Println(young1.Speak(think))

	var young2 People = YoungMan{}
	think = "bitch"
	fmt.Println(young2.Speak(think))

	var old1 People = &OldMan{}
	think = "bitch"
	fmt.Println(old1.Speak(think))

	var old2 People = OldMan{}
	think = "bitch"
	fmt.Println(old2.Speak(think))
}
```

**Q11答案**

<details>
<summary>点我显示</summary>

```
young2 = YoungMan{} 这一行会编译不通过
```
</details>

**Q11说明**

<details>
<summary>点我显示</summary>

```
- 仅young2编译不通过。原因是，指针类型的对象，可以同时调用结构体类型的方法，也可以调用指针类型的方法。但结构体类型的对象，只能调用结构体类型的方法。代码中young2是结构体类型对象，但YoungMan的speak方法是指针类型的方法，不可被结构体类型的对象识别，导致会认为缺少接口实现。
```
</details>

#### Q12：初始化

下面代码会输出什么

```
package main

import "fmt"

type Param map[string]interface{}

type Show struct {
	Param
}

func main() {
	s := new(Show)
	s.Param["RMB"] = 10000
	fmt.Println(s)
}
```

**Q12答案**

<details>
<summary>点我显示</summary>

```
panic: assignment to entry in nil map
```
</details>

**Q12说明**

<details>
<summary>点我显示</summary>

```
- 错误很简单，Param没有初始化，默认为nil。对nil赋值会报错
- 另外注意，Param没有额外命名变量名，这样写是许可的。和 param Param 一样。
- 另外，s 初始化使用的是new(Show)，等同于 s := &Show{}，所以此时s是个指针式对象。而一般常见的代码写法是 s := Show{}，这时的 s 是结构体对象。 和C不同，Go中new的对象不需要手动释放。
```
</details>

#### Q13：Json

下面代码会输出什么

```
package main

import (
	"fmt"
	"encoding/json"
)

type People struct {
	name string `json:"name"`
}

func main() {
	js := `{
		"name":"11"
	}`
	var p People
	err := json.Unmarshal([]byte(js), &p)
	if err != nil {
		fmt.Println("err: ", err)
		return
	}
	fmt.Println("people: ", p)
}
```

**Q13答案**

<details>
<summary>点我显示</summary>

```
people:  {}
```
</details>

**Q13说明**

<details>
<summary>点我显示</summary>

```
- people 结构中属性值为空的原因是， name 的首字母小写了。要符合正常预期，只需要改成下述代码即可
	Name string `json:"name"`
```
</details>

#### Q14：格式化输出

下面代码会输出什么

```
package main

import "fmt"

type People struct {
	Name string
}

func (p *People) String() string {
	return fmt.Sprintf("print: %v", p)
}

func main() {
	p := &People{}
	p.String()
}
```

**Q14答案**

<details>
<summary>点我显示</summary>

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```
</details>

**Q14说明**

<details>
<summary>点我显示</summary>

```
- 会输出堆栈溢出的原因是自我无限递归。因为 print(%v) 格式化生成字符串时，默认会调用自身的 String() 方法。
```
</details>

#### Q15：Channel

下面代码会输出什么

```
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int, 1000)

	go func() {
		for i := 0; i < 10; i++ {
			ch <- i
		}
	}()

	go func() {
		for {
			a := <-ch
			fmt.Println("a: ", a)
		}
	}()

	close(ch)
	fmt.Println("ok")
	time.Sleep(time.Second * 10)
}
```

**Q15答案**

<details>
<summary>点我显示</summary>

```
ok
a:  0
a:  0
panic: send on closed channel
```
</details>

**Q15说明**

<details>
<summary>点我显示</summary>

```
- 主线程过早的关闭channel，会导致写入channel时出错。
- 但主要注意的是，读取已经关闭的channel不会出错， <- channel 会持续读取到默认值（例如int类型的 channel 会读取到0），此时，需要用如下代码做检查。用 ok 来标示channel是否已经关闭。
		for {
			a, ok := <-ch
			if !ok {
				fmt.Println("close")
				return
			}
			fmt.Println("a: ", a)
		}
```
</details>
