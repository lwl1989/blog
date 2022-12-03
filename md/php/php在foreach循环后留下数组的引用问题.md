CreateTime:2019-09-27 17:59:58.0

# php在foreach循环后留下数组的引用问题

今天写代码的过程，发现老代码中出现了这种引用，粗看是没什么问题的，但是，来个demo吧。
```
foreach ($krs_info as &$item) {
            $item['title'] = $this->getEncryptKrInfo($item['title'], array_merge($user_ids_visible, [$item['user_id']]) , isset($params['user_id']) ? $params['user_id'] : 0);
        }
```


###  demo

```
<?php
  
$arr = [1,2,3,4];

foreach($arr as &$v) {
        //todo: do any more
}

var_dump($arr);
echo PHP_EOL;

foreach($arr as $v) {
        echo $v.PHP_EOL;
}
```
执行结果是什么呢？

```
array(4) {
  [0]=>
  int(1)
  [1]=>
  int(2)
  [2]=>
  int(3)
  [3]=>
  &int(4)
}

1
2
3
3
```
？？最后一位是3？


### 原因 

解释:

1. $v没有释放引用
2. 第二次循环开始时，第一次赋值将arr[0]赋值到了$v，原有arr[3]所占的地址空间的值变为了1，此时数组就是[1,2,3,1]。
3. 以此类推第一次赋值2->v，第二次赋值即3->v，第三次赋值即3->v，
4. 所以最终结果为[1,2,3,1]。

循环4次  数组的变化分别是:

```
1->    [1,2,3,1]
2->    [1,2,3,2]
3->    [1,2,3,3]
4->    [1,2,3,3]
```


### 解决方案

1. 尽量避免使用引用(加入key,推荐array_map来做子元素操作)

2. 引用使用了注意释放(unset)

