CreateTime:2016-07-25 15:38:14.0

## GOLANG
入门，废话不说，要知道它是什么，自己百度谷歌就OK的啦

## 首先，安装环境
本人的测试环境是centos7
下载编译后版本加入到环境变量即可
GO有个坑的地方是对gopath的设置，类似于JAVA的CLASS_PATH,但是针对每个项目得重新设置

## 编译工具
本人用的编译器是IDEA，对头，就是JAVA的那个
下载IDEA插件GOLANG，重起即可创建GO项目，具体使用，百度即可

## 语法
### 变量
GO语言可以使用多种方式定义变量，也可以直接使用赋值语句初始化变量
Golang的类型包含：
基本类型
    bool,int8-64,uint8-64,float32-64,complex64-128,string,rune,error
复合类型
    pointer,array,slice,map,chan,struct,interface
for example:
```Golang
//定义
var v1 int
var v2 string
var v3 [10]int //数组
var v4 []int   //切片
var v5 struct{
	f int
}
var v6 *int //指针
var v7 map[string]int
var v8 func(a int) int

var (
	v11 int
	v22 int
	v33 string
)
var v34 int = 33
var v45 = 44
	v56 := 33
	fmt.Println(v1,v2,v3,v4,v5,v6,v7,v8,v11,v22,v33,v34,v45,v56)
	
```
打印结果：
```
0  [0 0 0 0 0 0 0 0 0 0] [] {0} <nil> map[] <nil> 0 0  33 44 33
```
可以看出未赋值但是定义的变量都被初始化给予值了
### 常量
```
const test string = "a333b"
const  (
	c0 = iota
	c1 = iota
	c2 = iota
)
const (
	a0 = 1<<iota
	a1 = 2<<iota
	a2 = 3<<iota
)
fmt.Println(c0,c1,c2,a0,a1,a2)
fmt.Println(test)
```
常量有个要注意的地方是:(//iota 的值每次遭遇const永远是从0开始，然后从每次加1)
打印结果：
```
0 1 2 1 4 12
a333b
```
###关于Swap
Golang对于变量的交换，可以很方便的进行多值交换了
i = t; i = j; j = t 
变成了 
i,j = j,i 
for example:
```
	a,b = b,a   //   b=>a  a=>b  就近交换
	i,j := 1,2  //   :=是按顺序赋值
	var e,f int
	e,f = 1,2  //   =号是就近交换
	fmt.Println("i:=",i,"j=:",j)
	fmt.Println("f:=",e,"f=:",f)
```
### 关于整型
```
	var i1 int8
	//i1 = 133  超出范围  报错  -128 ~ 127
	i1 = 123

	var i2 uint8   //0-255  无符号整型 同  byte类型
	i2 = 235
	fmt.Println(i1,i2)

	var i3  int16
	i3 = 32767
	i4 := 32767
	fmt.Println(reflect.TypeOf(i4))
	i4 = 9223372036854775800  //question  为什么推导的类型不是 int64
//	i4 = 18446744073709551611 // question 之前推导的是int 64? 不能转换为uint64
	fmt.Println(i3,reflect.TypeOf(i4))
```

今天就到这了，该干别的去了 ，明天继续