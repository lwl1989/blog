CreateTime:2021-09-22 18:33:08.0

# 坑1，赋值

```
s1 = []int{1,2,3,4,5}
s2 := s1
s2 = append(s2, 1)
//s1[5]是什么？
```

由于切片是引用类型，首地址都一样，因此对当切片没有被扩容的时候，会影响之前的对象。如果扩容了，就不会影响了

所以引入了copy
```
s1 = []int{1,2,3,4,5}
s2 := make([]int,3)
copy(s2, s1)
s2 = append(s2, 1)
//s1[5]是什么？
```
# 坑2，参数传递

```
func sliceModify(slice []int) {
    slice = append(slice, 6)
}
func main() {
    slice := []int{1, 2, 3, 4, 5}
    sliceModify(slice)
    fmt.Println(slice)
}
```

返回的没变（虽然slice是引用类型，但是，他是作为值传递，因此，方法内部的slice改变了但是外部没有），这个设计太那啥了，可以正确跑出效果的版本如下：
```
func sliceModify(slice *[]int) {
    *slice = append(*slice, 6)
}
func main() {
    slice := []int{1, 2, 3, 4, 5}
    sliceModify(&slice)
    fmt.Println(slice)
}
```
# 坑3，多维数组

```
s1 := make([][]int,5)
s2 := make([][]int,3)
s1[0] = []int{1,2,3}
copy(s2, s1)
// s2[0]  what?
```

打印出来的s2[0] 居然也是1，2，3

为什么呢？

因为slice是引用类型，而copy只对元素进行复制，并不会管你是什么类型，此处大坑，要记住避免，否则很难排查。