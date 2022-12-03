CreateTime:2020-12-03 17:56:06.0

> emm，发觉自己没有系统的学习。都是有需求来临时调研技术，从今天起每天抽出时间系统化学习Go.

### 类型
Go是静态类型语言，运行期间不能改变类型

### 定义

1. 可以使用var 关键字
2. 可以使用 := 让系统推导类型

example:

```
var a int
a := 0
是一致的
```

但是要注意一点

```
var a int 
f := func () {
    a:=1 //此时是局部变量（没进行参数传递）
}
f()
此时a的值不会变
```

更灵活的语法：
```
var x , y ,z int
x,y,z := 1,float64(12313),"dassad" //动态类型推导，不建议这么写，增加阅读难度
```

### 特殊占位符 _

_用于将结果忽略的位置
```
func test() (int, string) {
   return 1, "abc"
}
func main() {
   _, s := test()
   println(s)
}
```


### 代码块定义

代码块内有新的变量生命周期
```go

s := "abc"
println(&s)

s, y := "hello", 20 // 重新赋值: 与前 s 在同⼀层次的代码块中，且有新的变量被定义。
println(&s, y) // 通常函数多返回值 err 会被重复使⽤。

{
    s, z := 1000, 30 // 定义新同名变量: 不在同⼀层次代码块。
    println(&s, z)
}
```


### 基本类型

|  类型  | 长度  | 默认值  | 说明  |
|  ----  | ----  | ---- |----  |
| bool  | 1 | false |  |
| byte  | 1 | 0 | uint8 之间可以相互转换 |
| rune  | 4 | 0 | unicode code point,int32 |
| int/uint  | 4或者8 | 0 | 32位长度4 64位长度8 |
| int8/uint8  | 1 | 0 | -128~127, 0~255 |
| int16/uint16  | 2 | 0 | -32768~32767,0~65535 |
| int32/uint32  | 4或者8 | 0 | 32位长度4 64位长度8, 21亿正负 |
| int64/uint64  | 8 | 0 |  |
| complex64  | 8 | 0 | 复数类型 |
| complex128  | 16 | 0 | 复数类型 |
| array  |  |  | 值类型 |
| struct  |  |  | 值类型 |
| string  |  | "" | 值类型 |
| slice  | 4或者8 | nil | 引用类型 |
| map  | 4或者8 | nil | 引用类型 |
| channel  | 4或者8 | nil | 引用类型 |
| interface  |  | nil | 接口类型 |
| function  |  | nil | 函数类型 |

#### new 和 make对引用类型的区别

引⽤类型包括 slice、map 和 channel。它们有复杂的内部结构，除了申请内存外，还需
要初始化相关属性。

new会计算引用类型大小，并为其分配初始值（默认值），并返回指针

make会被编译期编译为具体创建函数，分配内存和初始化成员结构,返回对象而非指针

```go
a:=[]int{0,0,0}
a[1] = 10
b := make([]int, 3) 
b[1] = 10 

c:=new([]int) //指针，但是只有头指针，可以append，但是不能直接使用
c[1]=10
```

### 常量

golang的常量只支持编译器可确定的类型。（数字、字符串、布尔）

不能定义在方法体内部

常量值还可以是 len、cap、unsafe.Sizeof 等编译期可确定结果的函数返回值。

重点： 编译期可确定的值(因为常量是不可写的，要在编译期确定结果，放入常量内存区域)

### 枚举

关键字 iota 定义常量组中从 0 开始按⾏计数的⾃增枚举值。

每一个const内iota都会从0开始

```go
const (
 Sunday = iota // 0
 Monday // 1，通常省略后续⾏表达式。
 Tuesday // 2
 Wednesday // 3
 Thursday // 4
 Friday // 5
 Saturday // 6
)

const (
 Sunday1 = iota // 0
 Monday1 // 1，通常省略后续⾏表达式。
 Tuesday1 // 2
 Wednesday1 // 3
 Thursday1 // 4
 Friday1 // 5
 Saturday1 // 6
)
```
