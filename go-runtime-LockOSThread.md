最近在学习kata源码时，看到runtime/virtcontainers/api.go开头有这么几行代码：
```
func init() {
  runtime.LockOSThread()
}
```
之前没用过runtime.LockOSThread函数，不过从函数名大概能猜出与操作系统县城有关，但具体是什么关系呢？我们先来看看该函数的注释：
```
// LockOSThread wires the calling goroutine to its current operating system thread.
// The calling goroutine will always execute in that thread,
// and no other goroutine will execute in it,
// until the calling goroutine has made as many calls to
// UnlockOSThread as to LockOSThread.
// If the calling goroutine exits without unlocking the thread,
// the thread will be terminated.
//
// All init functions are run on the startup thread. Calling LockOSThread
// from an init function will cause the main function to be invoked on
// that thread.
//
// A goroutine should call LockOSThread before calling OS services or
// non-Go library functions that depend on per-thread state.
```
简单转述一下该注释：如果某个goroutine使用了runtime.LockOSThread()函数，那么这个goroutine会连接到当前操作系统的线程，且该goroutine会一直在这个线程中执行，并保证没有其他goroutine会在这个线程中执行，也就是该goroutine独享一个线程。

goroutine独享线程可以使用在什么场景？又会有什么优势呢？举一个比较常见的场景：在“生产者-channel-消费者”模型中，生产者把消息放入channel，消费端起一个goroutine读取channel中的消息并做后续处理，例如分发给其他goroutine。此时处理接收消息的goroutine看起来会比其他goroutine更重要一些，我们当然也希望调度系统能给它分配更多的CPU时间片。但是在go的GPM调度模型中，用户创建的goroutine是公平调度的，假设服务中有几十上百万的goroutine在运行，上述处理消息的goroutine要与它们平等竞争显然不是特别理想。Talk is cheap, show me the code，下面我们来看看这种场景下的常规代码：
```
package main

import (
  "fmt"
  "os"
  "runtime"
  "time"
)

//启动5W个goroutine消耗CPU
func startSomeGoroutine() {
  var workload = func(){
    var count int
    for {
      count++
      if count >= 100000 {
        count = 0
        //让出CPU
        runtime.Gosched()
      }
    }
  }
  
  for i := 0; i < 50000; i++ {
    go workload()
  }
}

//启动20个goroutine模拟生产端往channel中写数据
func producers(ch chan struct{}) {
  for i := 0; i < 20; i++ {
    go func() {
      for {
        ch <- struct{}{}
      }
    }
  }
}

//消费端，读取一千万个数据
func consumer(ch chan struct{}) {
  for i := 0; i < 10000000; i++ {
    <-ch
  }
}

func main() {
  //10W 缓冲区的channel
  var ch = make(chan struct{}, 100000)
  
  //启动一些goroutine消耗CPU
  startSomeGoroutine()
  
  //生产者往channel中不断写数据
  producers(ch)
  
  go func() {
    tm := time.Now()
    consumer(ch) //消费者消费数据
    fmt.Println("cost: ", time.Now().Sub(tm))
    os.Exist()
  }()
  
  select{}
}
```
处理消息的goroutine与其它goroutine平等竞争，得到耗时结果：
```
cost: 12.0801537s
```
接下来在main函数中把runtime.LockOSThread()加上，使得该goroutine独享一个线程：
```
func main() {
  //10W 缓冲区的channel
  var ch = make(chan struct{}, 100000)
  
  //启动一些goroutine消耗CPU
  startSomeGoroutine()
  
  //生产者往channel中不断写数据
  producers(ch)
  
  go func() {
    runtime.LockOSThread()
    tm := time.Now()
    consumer(ch) //消费者消费数据
    fmt.Println("cost: ", time.Now().Sub(tm))
    runtime.UnlockOSThread()
    os.Exist()
  }()
  
  select{}
}
```
得到线程独享耗时结果：
```
cost: 933.9837ms
```
使用上述代码多次测试，线程独享比普通的公平调度有几倍到十几倍的性能优势。对于一些CPU密集的场景，让某个goroutine独享一个线程可以大幅提升性能。一些外部库像OS X Cocoa、OpenGL、SDL等，在某些场景需要在一个线程中完成连续的调用，这时候使用runtime.LockOSThread()也能很好的对接。
通过本篇文章，我们初步学习到了runtime.LockOSThread()函数可以使goroutine独享一个线程并带来性能的提升这一个小知识点。文章末尾留下个小问题：如果某个goroutine通过调用runtime,LockOSThread()函数独享了一个线程，那么这个goroutine创建的子goroutine是否会继承这个goroutine独享线程的特性？
