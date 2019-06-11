Q：Go中如何获取命令行参数？
A：使用os库的OS.Args，或者flag库。后者更加强大，有默认值，提示信息等支持。

Q：Go中如何对结构体struct对象进行拷贝赋值？
A：使用unsafe进行指针赋值（高效，但不安全）；使用反射赋值（安全，但很低速）；Json序列化后再反序列化（更慢，但很有想法）

Q：以下代码会输出什么，为什么
func main() {
    runtime.GOMAXPROCS(1)
    wg := sync.WaitGroup{}
    wg.Add(10)

    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println("i: ", i)
            wg.Done()
        }()
    }

  for i := 0; i < 5; i++ {
      go func(i int) {
          fmt.Println("i: ", i)
          wg.Done()
      }(i)
  }
  wg.Wait()
}
A：会输出 5555501234
因为第一个for循环，会极快的速度创建完5个go协程。此时i为5，所以输出的全是5。
第二个 for 循环，则因为每次都会将 i 的值拷贝一份传给 goroutine，得到的 i 不同，输出也会不同。

Q：如果尝试向一个已经关闭的channel进行数据读取，是否有问题？如果尝试向一个已经关闭的channel进行数据写入，是否有问题？
如果尝试向一个为nil的channel进行数据读取，是否有问题？如果尝试向一个为nil的channel进行数据写入，是否有问题？
若有问题，该如何解决？
A：向一个已经关闭的channel进行数据读取没有问题。向一个已经关闭的channel进行数据写入会引发panic。解决方法是写入时，做select..case done检查。
向一个为nil的channel进行数据读取或数据写入都会引发deadlock永久阻塞错误。

Q：如果使用标准HTTP库发起请求，获得一个无数据的空响应，此时是否需要手动关闭响应体？
A：空响应也必须手动 defer resp.Body.Close() 进行关闭，不然会出现内存泄露。

Q：如何对不确定格式的json进行解析？例如：[]byte(`{"status":200, "tag":"one"}`), []byte(`{"status":"ok", "tag":"two"}`)
A：使用 json.RawMessage 原生数据类型进行解析。 var result struct { Status     json.RawMessage `json:"status"` }

Q：以下代码会输出什么？为什么？
func main() {
	var i = 1
	defer fmt.Println("result: ", func() int { return i * 2 }())
	i = 2
}
A：输出2。因为defer参数在声明时就会求出具体值并入栈，而非执行时才求值。

Q：简述Golang的GC机制和其他GC相较的优缺点。
A：三色标记法。比常见的引用计数相比，优点是：内存碎片较少。缺点是：需要STW（stop the world）进行专门时间的内存释放。

Q：为什么Golang编译的exe可执行文件比C++编译的exe可执行文件通常大一些？
A：因为golang编译出的exe可执行文件内包含了runtime本身。用户代码都通过runtime与系统OS kernal交互。所以golang跨平台比较容易，但不同平台的可执行文件是不同的。

Q：一般你会怎么进行golang的BUG调试和性能问题解决？
A：用panic调用栈。 使用pprof输出性能分析。 做单元测试时进行benchmark。 使用go run/build -race做竞争检测。
