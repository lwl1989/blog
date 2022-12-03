CreateTime:2016-08-09 17:56:39.0

##总结一版
经过试验，我们发现了两个问题：
    1.当用户关系发生更改的时候，动态列表无法实时更新。（属于业务需求）
    2.实时性不强，容易出现错误数据
对上述问题我们进行了业务逻辑的更改。

##修改点
1.增加用户上一次拉取时间标记，由客户端传入
2.增加用户第一次发布动态时间标记
3.增加用户关系更改时间标记

主要是将用户关系的更迭能通知到timeline了。

##实现

 $last 为客户端传入的拉取时间
共有几个分支：

1.用户更迭时间大于拉取时间  
   
     移除timeline   读取实时动态
  2.用户更迭时间小于拉取时间 且  timeline不存在   
  
     读取实时动态   异步通知更新timeline
  3.用户更迭时间小于拉取时间  且  timeline存在  且  拉取时 间大于 timeline的最新数据缓存时间

     读取实时动态  异步通知更新timeline  //太新的数据有延迟 没法缓存

4.用户更迭时间小于拉取时间  且  timeline存在  且  拉取时间小于 timeline的最新数据缓存时间

        **读取实时动态  //太老的数据不缓存**
 5.用户更迭时间小于拉取时间  且  timeline存在  且 拉取时间在timeline时间段中

         **  读取timeline **

![输入图片说明](https://static.oschina.net/uploads/img/201608/09180253_8eMk.png "在这里输入图片标题")
##代码
```
		$updateTimeLine = false;

		if($followerUpdate > $last) {  
			$timeline->remove();
			$response = $this->_online($followers, $last, $limit, $order);
		} else {
			if(!$timeline->exists()) {
				$response = $this->_online($followers, $last, $limit, $order);
				$updateTimeLine = true;
			}else {
				$maxTime = $timeline->max();
				$maxTime = $maxTime['score'];
				$minTime = $timeline->min();
				$minTime = $minTime['score'];
				if($last>$maxTime) {
					$response = $this->_online($followers, $last, $limit, $order);
					$updateTimeLine = true;
				} else if($last < $minTime) {
					$response = $this->_online($followers, $last, $limit, $order);
				} else {
					$response = $timeline->rangeMax($last,$limit);
					$count = count($response);
					if($count != $limit) {
						$onlineResponse = $this->_online($followers, end($response),$limit-$count,$order);
						$response = array_merge($response, $onlineResponse);
					}
				}
			}
		}
```
如此很好的解决了实时的问题