CreateTime:2020-09-14 16:28:54.0



可能大家都在const中定义过原子数。const自带一个原子性自增的关键字iota

```
package main
 
import "fmt"
 
func main() {
    const name  = "BeinMenChuiXue"
    fmt.Println(name)
 
    const (
        homeAddr = "earth"
        nickName = "WeiLaiShiXuePai"
    )
    fmt.Println(homeAddr, nickName)
 
    const (
        zero = iota
        one
        two = iota
        three
    )
    fmt.Println(zero, one, two, three)
 
    const (
        first = iota
        second
    )
    fmt.Println(first, second)
 
    const (
        student = "Golang"
        teacher
        parents = "S"
        children
    )
    fmt.Println(student, teacher, parents, children)
}
```

