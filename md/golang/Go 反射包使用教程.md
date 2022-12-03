CreateTime:2020-01-22 16:43:22.0

# 引子

几乎所有高级语言都有反射功能，以用于语言的灵活性，实现各种封装。

Java(java.lang.reflect):
```
//获得Person的Class对象
Class<?> cls = testJavaSE.Person.class;//Class.forName("testJavaSE.Person");
//创建Person实例
Person p = (Person)cls.newInstance();
//获得Person的Method对象,参数为方法名,参数列表的类型Class对象
Method method = cls.getMethod("eat",String.class);
//invoke方法，参数为Person实例对象，和想要调用的方法参数
String value = (String)method.invoke(p,"肉");
//输出invoke方法的返回值
System.out.println(value);
```

Php(Reflection)
```
$reflect = new \\ReflectionClass();  
if($reflect->hasMethod("eat")) {  
  echo $reflect->getMethod("eat")->invoke($reflect->newInstance(), "php");
}
```

可以看到，我们可以很容易实现灵活的功能。那么go作为一门语法约束较为严格的语言，如果不支持反射的话，真的会很难大规模普及，所以官方还是提供了一个灵活的反射包（ps: 虽然不太好用）

# go 反射包

go反射包是存在与内核里面的

	GOROOT/src/reflect/*

这里要说明一点，go不是一门纯正的面向对象语言，所以反射包的涉及也略有不同。

**反射由reflect包提供支持，主要方法：**

-   func TypeOf ( i interface{} ) Type：

如 reflect.Typeof(x) ，形参x被保存为一个接口值并作为参数传递（复制），方法内部会把该接口值拆包恢复出x的类型信息保存为reflect.Type并返回

-   func ValueOf ( i interface{} ) Value：

如 reflect.ValueOf(x) ，形参被保存为一个接口值并作为参数传递（复制）, 方法内部把该接口值的值恢复出来保存为reflect.Value并返回；

### type

Type 接口：可以表示一个Go类型

Kind()将返回一个常量，表示具体类型的底层类型

Elem()方法返回指针、数组、切片、map、通道的基类型；

可用反射提取struct tag，还能自动分解，常用于ORM映射、数据验证等；

辅助判断方法Implements()、ConvertibleTo()、AssignableTo()

example(判断一个结构体是否包含某个属性):
```
t := reflect.TypeOf(s)
if _, ok := t.FieldByName("Name"); ok {
   return true
}
return false
```

### value

Value 结构体：可以持有一个任意类型的值
```
调用 Value 的 Type() 将返回具体类型所对应的 reflect.Type（静态类型）
调用 Value 的 Kind() 将返回一个常量，表示具体类型的底层类型
Interface方法是ValueOf方法的逆，它把一个reflect.Value恢复成一个接口值，把Value中保存的类型和值的信息打包成一个接口表示并返回：
通道类型的反射对象：有TrySend()、TryRecv()方法
IsNil()方法判断反射对象保存的值是否为nil
```

example(反射一个普通的数值):
```
var x int64 = 0
func getInterfaceValueInt64(x interface{}) int64 {
	//这里基础类型已经实现了封装
	//i64 := reflect.ValueOf(x).Int()
	i64 :=  reflect.ValueOf(x).(int64)
}
d: =getInterfaceValueInt64(x)
fmt.Println(d)
```

那么复杂类型如何处理呢？继续代码举例吧：

example(array or slice):
```
func SliceBuildWithInterface(v interface{}) reflect.Value {  
   typ := reflect.TypeOf(v) // 获取类型
   sliceValue := reflect.MakeSlice(reflect.SliceOf(typ), 0, 0)//生成value对象

   slice := reflect.New(sliceValue.Type()) //构建切片
   slice.Elem().Set(sliceValue) //设置切片值

   return slice
}
```

# 为什么需要反射呢？

-   功能更强大
-   更安全，防止直接调用没有暴露的内部方法
-   可维护，直接写字符串是硬编码