CreateTime:2020-12-08 17:04:00.0

> 前几天很尬，被人问redis中怎么去拉取一个类型的key。

## keys

文档：

	[keys command](https://redis.io/commands/keys)

我是这么回答，拉取key即可，还能用正则。

然后，就被深入问，如果碰到大量数据会出现什么问题，要怎么解决。

这就尴了个尬了，我记得可以用批处理脚本实现，但是具体是哪天命令我给忘记了。

之后便一直都在忙。然后在忙的途中，正好在使用工具。

我就发现了一个神奇的东西，原来是你啊，小宝贝儿。我想起来了。

![](https://oscimg.oschina.net/oscnet/up-6249bd59b6f1f08daf0f589ebccc1d89d1b.png)

## scan

文档：

	[scan command](https://redis.io/commands/scan)

搜达，那其实对于大批量的key，就是对游标进行遍历即可。（还是要用脚本实现滴）

## 拓展

> Available since 2.8.0.

> Time complexity: O(1) for every call. O(N) for a complete iteration, including enough command calls for the cursor to return back to 0. N is the number of elements inside the collection.

> The SCAN command and the closely related commands SSCAN, HSCAN and ZSCAN are used in order to incrementally iterate over a collection of elements.

> SCAN iterates the set of keys in the currently selected Redis database.
> SSCAN iterates elements of Sets types.
> HSCAN iterates fields of Hash types and their associated values.
> ZSCAN iterates elements of Sorted Set types and their associated scores.

看文档可以看到，scan命令对其他数据结构也进行了支持（突然我发现我之前用过了scan....我这个脑壳啊 太愚钝了）。

举一反三则可以得到，其他数据结构内，若想对集合或者map进行遍历，也可以用 *scan.


## 后记

很多用过的东西，没有及时归类，以至于当要想的时候要再次去查询。

以后要多多对自己的记忆进行分类式归档存储。