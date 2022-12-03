CreateTime:2015-10-18 20:26:11.0

将数组$arr分割为n个数组,并存放到一个二维数组中  返回值 二维数组
第三个参数表示 是否保留原来的下标
```
$arr = array(
    "key1" => "value1",
    "key2" => "value2",
    "key3" => "value3",
    "key4" => "value4",
);
array_chunk($arr,2,true);
array_count_values($arr)    统计$arr中值出现的次数

// select count(user) from  t  group by user; 同意义
```



###取数组差/交集
```
array_diff($arr1,$arr2,$arr3...)  //（只比较值）  $arr1-$arr2-$arr3-...
//  ['A','B','C'] - ['C','B','D']  =>  ['A']
array_diff_assoc   //比较下标和值

array_diff_key  //比较key(下标)
array_diff_ukey($arr1,$arr2,$arr3...,function)
array_diff_uassoc($arr1,$arr2,$arr3...,function)


//有=>  array_udiff系列为带索引       求差集
//有=>  array_uintersect()系列为     求交集
//有=>  array_intersect()系列为带索引 求交集
```



###一般内容操作

**用给定的值填充数组。**
给$arr指定下标起给定值   并且追加数量N个  $arr=array_fill(start,num,value);
```
print_r($arr1=array_fill(5,3,"ss"));
```

**用值将数组填补到指定长度**
相当于对数组的初始化  长度为num  ，加入数组已经有值存在  追加长度到 num
```
array_pad($arr,num,value);   
```


**在数组中搜索给定的值，如果成功则返回相应的键名。**
```
array_search(value,$arr,strict)   指定strict(true,false) 则检验数据类型
```	
****
```
array_sum($arr)	计算数组中所有值的和。
array_product($arr)  计算数组的值的乘积
array_rand($arr)      取数组随即值
array_reverse($arr)   反转数组
array_shift($arr)     删除数组的第一个元素  返回他的值
array_slice($arr,offset,length,preserve)  由offset截取长度为length的长度的值 指定preserve保留下标
array_splice($arr,offset,length,$arr1)    在$arr中由offset截取长度为length的长度的值 由 arr1替代
```

**函数从数组中把变量导入到当前的符号表中**
```
extract(array,extract_rules,prefix)

$a = 'Original';
$my_array = array("a" => "Cat","b" => "Dog", "c" => "Horse");
extract($my_array);
echo "\$a = $a; \$b = $b; \$c = $c";
输出：
$a = Cat; $b = Dog; $c = Horse
```
**检测内容是否存在**
```
in_array(value,array,[type])
```




###回调系列
在func函数中对$arr数组元素进行遍历  并返回相应想要的 （筛选数组） func返回的值是BOOLEAN
```
array_filter($arr,func);  
```

**将回调函数作用到给定数组的单元上**
func返回的string替代了原来的值   返回值是一个数组

```
array_map(func,$arr1,$arr2,$arr3......);
```

**用回调函数迭代地将数组简化为单一的值**
 将数组按照func中的方法进行加工  返回字符串，加入指定了inital第一个连接符为inital ,结果是将所有节返回值进行拼接      
```
array_reduce($arr,func,inital) 
```


array_walk(array,function,userdata...)
function有3个参数，前2个是必写，$key,$value，第三个是可选。
```
function myfunction($value,$key) 
{
echo "The key $key has the value $value<br />";
}
$a=array("a"=>"Cat","b"=>"Dog","c"=>"Horse");
array_walk($a,"myfunction");


//array_walk_recursive 递归调用
```


还有排序系列  就不看了。

