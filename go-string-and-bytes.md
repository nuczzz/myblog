在使用go语言编程过程中，string和bytes([]byte)是我们常见的两种数据类型，原生go使用强制类型转换及可支持这两种数据类型的转换：
```
var str = "string"
var bytes = []byte{'b', 'y', 't', 'e', 's'}
s2b := []byte(str)
b2s := string(bytes)
```
看到这里，你也许会觉得这篇文章应该就差不多完了，不就是string和[]byte的类型转换吗？别急，先来看点有意思的东西。我们先来写两个简单的关于string-to-bytes和bytes-to-string的性能测试函数：
```
package main

func BenchmarkNormalString2Bytes(b *testing.B) {
	var bytes []byte
	_ = bytes
	
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		bytes = []byte("test")
	}
}

func BenckmarkNormalBytes2String(b *testing.B) {
	var str string
	_ = str
	
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		str = string([]byte{'t', 'e', 's', 't'})
	}
}
```
执行性能测试命令（注意加上-benchmem参数查看内存分配情况）：
```
# go test -run=NONE -bench="^BenchmarkNormal(String2Bytes|Bytes2String)$" -benchmem
```
得到测试结果（注意最后两列）：
```
BenchmarkNormalString2Bytes-4  100000000  30.2 ns/op  4 B/op  1 allocs/op
BenchmarkNormalBytes2String-4  100000000  21.0 ns/op  4 B/op  1 allocs/op
```
有意思的事情来了。我们只是做了一个简单的string与[]byte的类型转换，竟然发生了一次4B的内存分配（最后一列为分配次数，倒数第二列为分配内存的大小，"test"字符串长度为4B）。当贝转换的数据长度不长时，可能感觉不到内存消耗的差异，但如果实在高并发场景下且被转换的数据是以M为单位呢？可能瞬间吃掉好几G的内存。那么，在上述简单的两个类型转换过程中为什么会出现内存分配呢？是否有方法可以对该操作做一些优化呢？我们还是先从一段简单的程序入手来剖析这个问题：
```
//main.go
package main

import (
	"fmt"
)

func main() {
	str := "hello world"
	bytes := []byte(str)
	fmt.Println(str, bytes)
}
```
为了更好的看出程序编译运行过程，我们在编译的时候加上禁优化（-N）和禁内联（-l, 小写L）参数：
```
# go build -gcflags "-N -l" -o main main.go
```
用gdb来查看程序运行过程的一些信息：
```
# gdb main
(gdb) l
1		pakcage main
2		
3		import (
4			"fmt"
5		) 
6		
7		func main() {
8			str：= "hello world"
9			bytes := []byte(str)
10		   fmt.Println(str, bytes)
(gdb) b 8
Breakpoint 1 at 0x47b7bf: file /root/gopath/src/zky/main.go, line 8.
(gdb) r
Starting program: /root/gopath/src/zky/main

Breakpoint 1, main.main() at /root/gopath/src/zky/main.go:8
8			str := "hello world"
(gdb) pt str
type = struct string {
	uint8 *str;
	int len;
}
(gdb) pt bytes
type = struct []uint8 {
	uint8 *array;
	int len;
	int cap;
}
(gdb)
```
通过上述gdb命令，我们可以看到string和[]byte（slice）在go的底层实现中是不同的，其中string的底层结构为（可以在src/runtime/string.go中找到定义）：
```
type = struct string {
	uint8 *str;
	int len;
}
```
slice的底层结构为（可以在src/runtime/slice.go中找到定义）：
```
type = struct []uint8 {
	uint8 *array;
	int len;
	int cap;
}
```
两种数据结构的第一个字段都是指向具体数据的数组指针，第二个字段是数据的长度，slice比string多一个int类型的cap字段。看到这咯，我们可以慢慢来分析为什么以爱是的程序会出现内存分配问题了。以string转[]byte为例：假设除湿"test"字符串的地址为&ptr1，也就是该string对象底层结构的str字段为&ptr1，当执行[]byte(str)时，go会创建一个slice对象，并将string底层的"test"数据复制过来，也就是说最后string对象和[]byte对象的数据并不是同一份，这就出现了一开始的内存分配问题。我们可以接着上边的gdb进行验证：
```
(gdb) x/2xg &str
0xc420037ed0:	0x00000000004a5cff	0x000000000000000d
(gdb) x/3xg &str
0xc420037f10:	0x000000c42000e290	0x000000000000000d
0xc420037f20:	0x0000000000000010
```
可以看到string对象底层数据地址是0x00000000004a5cff，而[]byte对象底层数据地址是0x000000c42000e290，显然不是同一个地址，当然也就不是同一份数据了。
既然原生的go在string和[]byte转换时会出现数据复制而出现重新分配内存现象，那么有没有办法不让它进行数据复制，仅使用同一份数据呢？答案是可以的，不过在这之前，先来回顾一下go的两种指针unsafe.Pointer和uintptr，可以分别用一句话来概括：
```
unsafe.Pointer:		能与任何类型指针转换，但不能做加减偏移操作
uintptr:			只能与unsafe.Pointer类型指针转换，但能做加减偏移操作
```
从上面gdb查看到的string和[]byte底层数据结构信息中。我们可以把string看做[2]uintptr，而[]byte看做[3]uintptr，于是string-to-bytes只需要构建[3]uintptr{ptr, len, len}，而bytes-to-string则更简单，直接转转指针类型，忽略掉cap即可。这样我们只是用了同一份底层数据，不会存在复制数据给新对象的动作，于是有了以下代码：
```
func String2Bytes(str string) []byte {
	x := (*[2]uintptr)(unsafe.Pointer(&str))
	h := [3]uintptr{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}

func Bytes2String(bs []byte) string {
	return *(*string)(unsafe.Pointer(&bs))
}
```
我们先通过benchmark新能测试来验证没有内存分配，编写如下代码：
```
package main

import (
	"testing"
	"unsafe"
)

func String2Bytes(str string) []byte {
	x := (*[2]uintptr)(unsafe.Pointer(&str))
	h := [3]uintptr{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}

func Bytes2String(bs []byte) string {
	return *(*string)(unsafe.Pointer(&bs))
}

func BenchamrkNormalBytes2String(b *testing.B) {
	var str string
	_ = str
	bytes := []byte{'t', 'e', 's', 't'}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		str = string(bytes)
	}
}

func BenchmarkPointerString2Bytes(b *testing.B) {
	var bytes []byte
	_ = bytes
	str := "test"

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		bytes = string2Bytes(str)
	}
}

func BenchmarkPointerBytes2String(b *testing.B) {
	var str string
	_ = str
	bytes := []byte{'t', 'e', 's', 't'}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		str = Bytes2String(bytes)
	}
}
```
来看一下性能测试结果：
```
# go test -run=NONE -bench="^Benchmark(Normal|Pointer)(String2Bytes|Bytes2String)$" -benchmem

BenchmarkNormalString2Bytes-2  30000000   39.5 ns/op  8 B/op  1 allocas/op
BenchmarkNormalBytes2String-2  50000000   36.3 ns/op  4 B/op  1 aloocas/op
BenchmarkPointerString2Bytes-2 500000000  3.37 ns/op  0 B/op  0 aloocas/op
BenchmarkPointerBytes2String-2 2000000000 1.47 ns/op  0 B/op  0 allocas/op
```
可以看到通过指针做string与[]byte数据类型转换，没有了内存分配过程，性能也有了十几倍的提升。以string-to-bytes为例，再来用gdb验证底层数据是用的同一份。先编写如下main.go:
```
package main

import (
	"fmt"
	"unsafe"
)

func String2Bytes(str string) []byte {
	x := (*[2]uintptr)(unsafe.Pointer(&str))
	h := [3]uintptr{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}

func main() {
	var str = "hello world"
	bytes := String2Bytes(str)
	fmt.Println(str, bytes)
}
```
编译后用gdb查看相关信息：
```
# go build -gcflags "-N -l" -o main main.go
# gdb main
(gdb) l 14
9			 x := (*[2]uintptr)(unsafe.Pointer(&str))
10			h := [3]uintptr{x[0], x[1], x[1]}
11			return *(*[]byte)(unsafe.Pointer(&h))
12		}
13
14		func main() {
15			var str = "hello world"
16			bytes := String2Bytes(str)
17			fmt.Println(str, bytes)
18		}
(gdb) b 17
Breakpoint 1 at 0x47b8ba: file /root/gopath/src/zky/main.go, line 17.
(gdb) r
Starting program: /root.gopath/src/zky/main

Breakpoint 1, main.main() at /root/gopath/src/zky/main.go:17
17			fmt.Println(str, bytes)
(gdb) x/2xg &str
0xc420037ed0	0x00000000004a5d9f	0x000000000000000d
(gdb) x/3xg &bytes
0xc420037f10	0x00000000004a5d9f	0x000000000000000d
0xc420037f20	0x000000000000000d
(gdb)
```
可以看到str和bytes的数据地址都是0x00000000004a5d9f，这样就验证了两个变量底层的数据是公用的同一份。
其实go的reflect包下（src/reflect/value.go）有定义这么两个结构体：
```
type stringHeader struct {
	Data uintptr
	Len int
}

type SliceHeader struct {
	Data uintptr
	Len int
	Cap int
}
```
如果对至真不是特别熟悉，我们可以考虑借助这两种结构体类型来完成string与[]byte的转换（代码如下），但是经测试这种实现方式在性能方面大约只有纯至真操作的一半。
```
func String2Bytes(str string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := reflect.SliceHeader {
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}
	return *(*[]byte)(unsafe.Pointer(&bh))
}
```
上面我们提到三种string与[]byte的转换方法：go原生类型转换、完全指针转换以及借助结构体进行数据类型转换，其中go原生类型转换需要重新申请一片内存空间做数据拷贝，而后两种方式，string与[]byte底层数据是同一份，省去了数据拷贝过程，在性能上有较大的提升。
