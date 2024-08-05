# 字符串

这一篇文档主要叙述了在 Go 语言中，字符串的相关处理。比如说求取字符串的长度、子串查询、字串统计、字串的分割、字串的连接、替换、开始和结尾字符的判断等。

### 获取字符串的长度

在 Go 语言中，获取字符串的长度要区分两种情况，第一种是纯英文字符，第二种是包含中文字符。

因为我们知道，如果是纯英文字符，使用 ASCII 码就可以涵盖，只需要一个字节即可。但是如果包含中文字符，怎需要使用 Unicode 编码字符集，常用的是 UTF-8 字符集，它是变长字符集。也就是说如果是字符 `a` 这样的简单字符，仍旧是采用一个字节，但是如果遇上中文就可能采用三个字节存储。

下面我们先求取纯英文字符串的长度:

```go
var str string = "Hello World"
fmt.Printf("%d\n", len(str))		// Output: 11
```

如果包含中文呢呢？就不对了，我们希望一个中文就是一个长度，如下示例:

```go
var str string = "Hello 世界"
fmt.Printf("%d\n", len(str))		// Output: 12
```

如果包含中文的话，就需要使用如下的形式来获取长度:

```go
var str string = "Hello 世界"
// 将字符串强行转为 []rune 类型, 然后使用 len 求取其数组元素个数
fmt.Printf("len : %d\n", len([]rune(str)))
// 使用 []int32 也可以，其实 rune 类型是 int32 的别名，所以两者等效
fmt.Printf("len : %d\n", len([]int32(str)))
```

### 字符串的内存结构

在了解了如何获取字符串的长度之后，我们来看看字符串在内存占用多少空间。下面的代码演示了如何获取字符串变量的内存空间占用：

```go
a := "Hello World"
b := "You are my sunshine!"
fmt.Printf("%d %d", unsafe.Sizeof(a), unsafe.Sizeof(b)) // 16 16
```

从上面的代码中我们可以看到，字符串变量不管内容是什么，空间占用是一样的，都是 16 个字节（64bit OS）。这是为什么呢？我们来看看源码中字符串是如何表示的，文件 `runtime/string.go` 中定义如下：

```go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

这个结构体有两个成员，`str` 指针指向的是一个 `byte`的数组，指针成员在 64bit 的操作系统上总是 8 个字节，另一个是一个 int64 记录字符串的长度，同样占用了 8 个字节。两个成员变量，不存在内存对齐，就构成这个结构体的在内存中的大小，16 字节。所以字符串变量的长度永远都是 16 字节。

我们再来深入探究一下，这个 `len` 变量中保存的数值的具体含义。这个结构体在外面是不能访问的，但是我们可以通过 `len` 内置函数来获取这个值:

```go
a := "Hello,小王"
fmt.Printf("%d", len(a)) // 12
```

可以看到，这个输出的结果是12，而不是  8。为什么呢？因为这个统计的是字符串的字节数。在 Go 语言中，采用的是 UTF-8 字符编码。 UTF 指的是 Unicode ，包括了全世界 159 中文字，总计 144679 个字符。而 UTF-8 是变长字符集，而比如`Hello,` 这些都是英文字符，所以长度都是 1 个字节，而中文字符数量更多，长度大多为 3 个字节。所以，`Hello,小王` 最终输出是 12 个字节。

除了使用 `len` 内置函数外，我们可以使用 `StringHeader` 这个结构体来获取，它被定义在`runtime/value.go` 这个文件中:

<img src="http://file-linker.oss-cn-hangzhou.aliyuncs.com/cZWkJ1KvMXGMxK9qfGWa.png" alt="img1"/>

这个结构和 `stringSTruct`是一样的，只是这个是可以在外面访问的。我们来获取 `Len`：

```go
a := "Hello,小王"
// 先将变量 a 的指针强转为 unsafe.Pointer 这个万能指针
// 然而再次强转为 StringHeader 指针
sh := (*reflect.StringHeader)(unsafe.Pointer(&a))
fmt.Println(sh.Len)		// 12
```

所以我们不能采用一般的方式遍历字符串，而是要采用 `for...range...`, 如下:

```go
a := "Hello,小王"
// 下面这种方式是不行的
for i := 0; i < len(a); i++ {
	// 输出: 72 101 108 108 111 44 229 176 143 231 142 139
	fmt.Print(a[i], " ")
}
fmt.Println()
// 应该采用下面的方式
for _, char := range a {
	// 输出: H e l l o , 小 王
	fmt.Printf("%c ", char)
}
```

> 关于 UTF-8 字符的编码以及解码的实现，可以参考源码中的 `runtime/utf8.go` 文件中的 `encoderune` 方法以及 `decoderunne` 方法。


### 字串的操作 {id="string_actions"}

在编程中，我们经常要在一个字符串中查询另一个字符串是否存在。关于字符串的操作，它们都包含在 `strings` 的包中，使用前需要先引入这个包:

```go
import "strings"
```

一些常见的字串操作。

```go
var str string = "Hello World"
// 返回是否包含字串
fmt.Printf("%v\n", strings.Contains(str, "World")) // Output: true
// 返回字串的位置
fmt.Printf("%v\n", strings.Index(str, "World"))   // Output: 6
// 返回是否以指定字串开头
fmt.Printf("%v\n", strings.HasPrefix(str, "Hello")) // Output: true
// 返回是否以指定字串结尾
fmt.Printf("%v\n", strings.HasSuffix(str, "World")) // Output: true
```

### 其他常用的操作 {id="actions"}

还有一些其他常见的操作，比如字符串的大小写转换:

```go
// 转换成大写
fmt.Printf("%v\n", strings.ToUpper(str))	// Output: HELLO WORLD
// 转换成小写
fmt.Printf("%v\n", strings.ToLower(str))	// Output: hello world
```

去除空格:

```go
// 去除前后空格
fmt.Printf("%v\n", strings.TrimSpace("  Hello World  ")) // Output: Hello World
// 去除左右空格
fmt.Printf("%v\n", strings.TrimLeft("  Hello World  ", " ")) // Output: Hello World
fmt.Printf("%v\n", strings.TrimRight("  Hello World  ", " ")) // Output:     Hello World
```

字符串和切片的相互转换:

```go
// 字符串和切片的相互转换
fmt.Printf("%v\n", strings.Split("Hello World", " ")) // Output: [Hello World]
fmt.Printf("%v\n", strings.Join([]string{"Hello", "World"}, " ")) // Output: Hello World
```

字符串的替换:

```go
fmt.Printf("%v\n", strings.Replace("Hello World", "World", "China", 1)) // Output: Hello China
fmt.Printf("%v\n", strings.ReplaceAll("Hello World", "o", "*")) // Output: Hell* W*rld 
```

### 格式化输出 {id="format"}

输出字符串只要有两个函数，分别是  `fmt.Println` 和 `fmt.Printf` ，前者可以输出换行，后者可以使用占位符来输出更复杂的组合，可读性也更强一些:

```go
name := "Tom"
fmt.Println("Hello, " + name)
fmt.Printf("Hello, %s\n", name)
```

这里需要说明的是，使用 `%v` 占位符，它可以根据变量的类型，以预定的形式输出。如果希望输出变量的类型，可以使用 `%T` 占位符。

## 总结 {id="summary"}

总的来说，Go语言中的字符串是一种常用的数据类型，提供了丰富的字符串操作函数和方法，使得字符串的处理更加方便和高效。