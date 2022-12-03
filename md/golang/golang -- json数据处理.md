CreateTime:2017-11-30 14:08:12.0

### 包介绍
官方提供了一个处理json的包（encoding/json）,导入即可使用

### 数据结构

json源于javascript的对象结构，golang中直接对应的数据结构，可是golang的map也是key-value结构，同时struct结构体也可以描述json。当然，对于json的数据类型，go也会有对象的结构所匹配。


结构 | go | json
---|---|---
字符串 | string | string
整型 | int* | number
浮点 |float或者double|number
数组 | slice | array
对象 | struct | object
布尔 | bool | bool
空值 | nil | null

### 如何将go的数据转换成json

#### 基础
假如使用的是官方的json库，需要2步:

1. 定义数据结构(定义结构体的时候，只有字段名是大写的，才会被编码到json当中。)
2. 调用json.Marshal生成数据

#### 简单结构
定义：
```
type Account struct {
        Email string
        password string
        Money float64
}
```
调用
```
 account := Account{
                Email: "我我我我aha@q.com",
                password:"qqqqqqqq",
                Money: 200.33,
        }

        rs, err := json.Marshal(account)
        if err != nil {
                fmt.Println(err)
        }

        fmt.Println(rs)
        fmt.Println(string(rs))
```

结果：

    [123 34 69 109 97 105 108 34 58 34 230 136 145 230 136 145 230 136 145 230 136 145 97 104 97 64 113 46 99 111 109 34 44 34 77 111 110 101 121 34 58 50 48 48 46 51 51 125]
    {"Email":"我我我我aha@q.com","Money":200.33}

观察结果，可以发现，英文字符和普通字符都被转码成ASCII的数值替代，而无法被ASCII标识的字符则使用了3个位置（比如中英文混编的“我我我我aha@q.com”字=>[34 230 136 145 230 136 145 230 136 145 230 136 145 97 104 97 64 113 46 99 111 109 34]），“230 136 145” 即代表 “我”字

#### 复合结构

go中个人理解是以类型为中心的语言。一个复杂的数据结构都是由多个简单的数据结构组合而成。

因此，简单的数据结构可以转，那么复合结构自然是没有问题。

定义：
```
type User struct {
        Name string  
        Age int 
        Roles []string  //slice
        Skill map[string]float64  //map
        Extra []interface{} //空类型
}
```

调用
```
 skill := make(map[string]float64)
        
        skill["python"] = 99.5
        skill["elkxkr"] = 90
        skill["ruby"] = 80.0
        
        extra := []interface{}{"ha",skill}
        user := User{
                Name:"战三",
                Age:28, 
                Roles:[]string{"Owner","Master"},
                Skill:skill,
                Extra:extra,
        }      
        
        rs1, err1 := json.Marshal(user)
        if err1 != nil {
                fmt.Println(err1)
        }       
        
        fmt.Println(rs1)
        fmt.Println(string(rs1))
```
结果
```
[123 34 78 97 109 101 34 58 34 230 136 152 228 184 137 34 44 34 65 103 101 34 58 50 56 44 34 82 111 108 101 115 34 58 91 34 79 119 110 101 114 34 44 34 77 97 115 116 101 114 34 93 44 34 83 107 105 108 108 34 58 123 34 101 108 107 120 107 114 34 58 57 48 44 34 112 121 116 104 111 110 34 58 57 57 46 53 44 34 114 117 98 121 34 58 56 48 125 44 34 69 120 116 114 97 34 58 91 34 104 97 34 44 123 34 101 108 107 120 107 114 34 58 57 48 44 34 112 121 116 104 111 110 34 58 57 57 46 53 44 34 114 117 98 121 34 58 56 48 125 93 125]
{"Name":"战三","Age":28,"Roles":["Owner","Master"],"Skill":{"elkxkr":90,"python":99.5,"ruby":80},"Extra":["ha",{"elkxkr":90,"python":99.5,"ruby":80}]}
```

#### 别名[json tag]

由于json世界通常是小写，而go只能导出大写的字段才会导出，因此，定义结构的时候提供别名的方式。

```
type Student struct {
    Name string `json:"name"`
}

可以理解成sql的  Name as name
```

当有多个字段的时候，可以只设置部分字段的别名，按需设置即可。

### 解码
按照通常的命名规则 Marsshal 是编码  解码则是Unmarsshal(data []byte, v interface{})


```
user := User{}
err  = json.Unmarshal(jsonByte, &user)
fmt.Println(user);
```
结果
```
{战三 28 [Owner Master] map[elkxkr:90 python:99.5 ruby:80] [ha map[elkxkr:90 python:99.5 ruby:80]]}
```
### 其他

json tag中还提供 


1.omitempty方式

比如
```
type Student struct {
    Name string `json:"name,omitempty"`
}
```

则name为空的时候不进行输出

2.string

```
type Student struct {
    Age    int `json:"age,string"`
}
```

则生成的是string字符串 
（age=3 => "age":"3"）

3.其他包

    github.com/panthesingh/goson  提供复合数据的二次处理  

### 参考

[goexample](https://gobyexample.com/json)

[人世间-Golang处理JSON（一）--- 编码](http://www.jianshu.com/p/f3c2105bd06b)