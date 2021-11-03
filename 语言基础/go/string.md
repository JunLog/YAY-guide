# string

> 神秘的字符串！！

## 字符集

### ASCII 码

一个`bit`有0和1两种状态，8个`bit`代表一个字节，有256个状态，`00000000`（代表0）到`11111111`（代表255）。数字可以通过值来表示，字符如何表示？

大写字母A如何在计算机中表示呢？给他指定一个数字编号`01000001`，当需要存储A时候就存储这个编号值，要读取时候，根据值通过映射关系找到这个字符。向这样去收录很多字符进行统一编号，得到一个字符编号表也叫做字符集。

上个世纪60年代，美国制定了一套字符编码，对英语字符与二进制位之间的关系，做了统一规定。这被称为 ASCII 码，一直沿用至今。ASCII 码一共规定了128个字符的编码，大写字母`A`是65（二进制`01000001`）。

### Unicode

英语用128个字符编码就够了，但是用来表示其他语言就显得很少了，亚洲国家的文字，使用的符号就更多了，汉字就多达10万左右。后面推出了GB2312（存储简体中文），有部分人需要繁体字，那么有推出了BIG5，不断地推出新的字符集。

因此，要想打开一个文本文件，就必须知道它的编码方式，否则用错误的编码方式解读，就会出现乱码。为了减少这种情况的产生，推出了一个统一的字符集Unicode，将世界上所有的符号都纳入其中。每一个符号都给予一个独一无二的编码，那么乱码问题就会消失。

Unicode 是一个很大的集合，现在的规模可以容纳100多万个符号。每个符号的编码都不一样。

> ❓：unicode规定了字符的二进制形式，但是没有规定这个二进制如何存储？不同的字符对应的二进制位数肯定也不一样。

计算机无法知道一串二进制数中三个字节表示的是一个字符而不是表示三个字符。如果都规定用文本中最大的字节作为一个字符来存储（**定长编码**），英文字母只需要一个字节，那么必然会导致英文字母前面有多个字节是0。这对于存储来说是极大的浪费，文本文件的大小会因此大出二三倍甚至更大，这是无法接受的。

这个编码问题没有解决，Unicode 在很长一段时间内无法推广。

### UTF-8

互联网的普及，强烈要求出现一种统一的编码方式。UTF-8 就是在互联网上使用最广的一种 Unicode 的实现方式。我们来看看UFT-8的编码规则是怎么样的？

UTF-8 最大的一个特点，就是它是一种**变长**的编码方式。它可以使用**1~4个字节**表示一个符号，**根据不同的符号而变化字节长度。**

UTF-8的编码规则：

1. 对于单字节的符号，字节的第一位设为`0`，后面7位为这个符号的 `Unicode` 码。因此对于英语字母，UTF-8 编码和 ASCII 码是相同的。
2. 对于`n`字节的符号（`n > 1`），第一个字节的前`n`位都设为`1`，第`n + 1`位设为`0`，后面字节的前两位一律设为`10`。剩下的没有提及的二进制位，全部为这个符号的 Unicode 码。

```
Unicode符号范围     |        UTF-8编码方式
(十六进制)          |              （二进制）
----------------------+---------------------------------------------
0000 0000-0000 007F [0,127] | 0xxxxxxx
0000 0080-0000 07FF [128,2047] | 110xxxxx 10xxxxxx
0000 0800-0000 FFFF [2048,65535]  | 1110xxxx 10xxxxxx 10xxxxxx
0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

> 举例

`严`的 Unicode 是`4E25`（`0100 1110 0010 0101`），根据上表，可以发现`4E25`处在第三行的范围内（`0000 0800 - 0000 FFFF`），因此`严`的 UTF-8 编码需要三个字节，即格式是`1110xxxx 10xxxxxx 10xxxxxx`。然后，从`严`的最后一个二进制位开始，依次从后向前填入格式中的`x`，多出的位补`0`。这样就得到了，`严`的 UTF-8 编码是`11100100 10111000 10100101`，转换成十六进制就是`E4B8A5`。

## 字符串（string）

Go的源文件总是按UFT-8进行编码，习惯上Go的字符串会按UTF-8解读。

> string在go中如何定义

方式一：

```go
func main(){
	var s string="123\t456\n"
	fmt.Printf(s)
}
```

同时在带双引号的字符串字面量中可以使用ASCII码中的转义符。

方式二：

```go
func main(){
	var s string=`q23`
	fmt.Printf(s)
}
```

采用``反引号的方式不会对字符串里面的转义符进行转义，但是可以创建多行的字符串。

> string底层结构

string需要一个起始地址，在内存中可以找到字符串的内容，在C语言中通过特定的字符`/0`来表示字符串结束,但是限制字符串不能出现`/0`字符，而是通过长度len来表示字符串结束。

**len长度不是字符个数，而是字节个数。**

![image-20211030162201466](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211030162201.png)

```go
struct String
{
        byte*   str;//首地址
        intgo   len;//长度
};
```

在java 和 C 语言中，字符串一般是由`char[]`数组定义，而go 采用`byte`数组，其实主要和go语言在创建之初并不想以ASCII码为中心，其采用`[]byte`的方式，使得在字符串接收时，不会出现乱码。同时go为更方便的处理非ASCII字符串时，定义了`rune`类型，获取一个`Unicode`字符。

rune类型作为int32类型的别名，同时byte式int8类型的别名。

```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```

证明：

```go
func main(){
	s:="yay世界"
	fmt.Println("字节输出")
	for i:=0;i<len(s);i++{
		fmt.Printf("%c\n",s[i])
	}
	fmt.Println("字符输出")
	for _,item:=range s{
		fmt.Printf("%c\n",item)
	}
}
```

![image-20211030154206313](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211030154206.png)

> string的特性

Go中的string被定义为**只读类型**。字符串在编程中经常会被使用到，只读可以保证数据的安全，减少编程的复杂度。**不允许修改字符串中的字符**。字符串变量是可以**共用底层字符串**实现的。

如果你想修改字符串内容，一种方法，**可以赋新值**，字符串并没有修改原本的内存对应内容，而是指向新的内存，另一种方法：**变量强制类型转换成字节slice**，就可以进行修改。

关于slice建议去看[slice讲解](./数组与slice.md)

## 参考文章

[字符编码笔记：ASCII，Unicode 和 UTF-8](https://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)

[go 语言string之解析](https://cloud.tencent.com/developer/article/1784866)

[Go语言标准库学习之strings——字符串格式化的利器](https://blog.csdn.net/random_w/article/details/107768992)

[十分钟搞清字符集和字符编码](http://cenalulu.github.io/linux/character-encoding/)

