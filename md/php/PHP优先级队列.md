CreateTime:2018-11-29 15:24:48.0

## 优先级队列

首先，我们要了解一下什么叫队列：

> 队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。

从定义来看，队列是无法更改顺序的线性集合。线性集合一般有几种规则：先进先出（队列）、先进后出（栈）。

优先级队列定义如下：

> 如果我们给每个元素都分配一个数字来标记其优先级，不妨设较小/较大的数字具有较高的优先级，这样我们就可以在一个集合中访问优先级最高的元素并对其进行查找和删除操作了。这样，我们就引入了优先级队列 这种数据结构。 优先级队列（priority queue） 是0个或多个元素的集合，每个元素都有一个优先权，对优先级队列执行的操作有（1）查找（2）插入一个新元素 （3）删除 一般情况下，查找操作用来搜索优先权最大的元素，删除操作用来删除该元素 。对于优先权相同的元素，可按先进先出次序处理或按任意优先权进行。

可以看到，优先级队列对队列进行了优化，从根本上已经改变了进出顺序（当然，若优先级一样的话，则于队列还是一样的处理方式）。

## PHP的实现方式

从php5.3起，内部已经实现了优先级队列的类：[SplPriorityQueue](http://php.net/manual/zh/class.splpriorityqueue.php "SplPriorityQueue")，注意，官方有一句话：

> The SplPriorityQueue class provides the main functionalities of a prioritized queue, implemented using a max heap.

可以看到，php的优先级队列是用大顶堆实现的算法，其初始化时间复杂度为O(n)，重排时间复杂度为O(logn)。

具体可以点链接查看，现在我们来看几个重要的方法：


#### compare
```
SplPriorityQueue::compare — Compare priorities in order to place elements correctly in the heap while sifting up
public int compare ( mixed $priority1 , mixed $priority2 )
```
比较优先级，以便在筛选时将元素正确地放置在堆中。

咦，好像JAVA的样子。

#### setExtractFlags

```
SplPriorityQueue::setExtractFlags — Sets the mode of extraction
public void SplPriorityQueue::setExtractFlags ( int $flags )
```
从字面意义上来看就是设置提取标志，参数有如下：
```
SplPriorityQueue::EXTR_DATA (0x00000001): Extract the data
SplPriorityQueue::EXTR_PRIORITY (0x00000002): Extract the priority
SplPriorityQueue::EXTR_BOTH (0x00000003): Extract an array containing both
```
分别为处理数据、优先级、两者都进行，怎么理解呢？来搞个demo吧
```
<?php

class TestPriority extends SplPriorityQueue {
    //值大的优先级大
	public function compare($priority1, $priority2) {
		if($priority1 == $priority2) return 0;
		return $priority1 > $priority2 ? 1 : -1;
	}
}

$p = new TestPriority();
$p->insert('a', 3);
$p->insert('b', 5);
$p->insert('c', 2);
$p->insert('d', 1);
$p->insert('e', 4);
$p->insert('f', 9);
$p->insert('g', 1);

$p->setExtractFlags(SplPriorityQueue::EXTR_BOTH);

if($p->count() > 0) {
	while($p->valid()) {
		$v = $p->current();
		var_dump($v);
		$p->next();
	}
}

$p->insert('a', 3);
$p->insert('b', 5);
$p->insert('c', 2);
$p->insert('d', 1);
$p->insert('e', 4);
$p->insert('f', 9);
$p->insert('g', 1);
$p->setExtractFlags(SplPriorityQueue::EXTR_DATA);

if($p->count() > 0) {
	while($p->valid()) {
		$v = $p->current();
		var_dump($v);
		$p->next();
	}
}

$p->insert('a', 3);
$p->insert('b', 5);
$p->insert('c', 2);
$p->insert('d', 1);
$p->insert('e', 4);
$p->insert('f', 9);
$p->insert('g', 1);
$p->setExtractFlags(SplPriorityQueue::EXTR_PRIORITY);

if($p->count() > 0) {
	while($p->valid()) {
		$v = $p->current();
		var_dump($v);
		$p->next();
	}
}
```

运行结果如下：

```
array(2) {
  ["data"]=>
  string(1) "f"
  ["priority"]=>
  int(9)
}
array(2) {
  ["data"]=>
  string(1) "b"
  ["priority"]=>
  int(5)
}
array(2) {
  ["data"]=>
  string(1) "e"
  ["priority"]=>
  int(4)
}
array(2) {
  ["data"]=>
  string(1) "a"
  ["priority"]=>
  int(3)
}
array(2) {
  ["data"]=>
  string(1) "c"
  ["priority"]=>
  int(2)
}
array(2) {
  ["data"]=>
  string(1) "d"
  ["priority"]=>
  int(1)
}
array(2) {
  ["data"]=>
  string(1) "g"
  ["priority"]=>
  int(1)
}
string(1) "f"
string(1) "b"
string(1) "e"
string(1) "a"
string(1) "c"
string(1) "d"
string(1) "g"
int(9)
int(5)
int(4)
int(3)
int(2)
int(1)
int(1)
```
很明显能够观察到值得变化。

#### 注意

执行处理之前一定要对长度进行判断，否则会出现如下错误：

```
PHP Fatal error:  Uncaught RuntimeException: Can't peek at an empty heap in /Users/wenglong11/Desktop/priority.php:30
```
