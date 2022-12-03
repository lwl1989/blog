CreateTime:2016-08-04 12:42:52.0

###用户关系
####用户角度
我的好友
我关注的
我封锁的
####粉丝角度
我的粉丝（关注我的）
封锁我的

其中我的好友和我关注的可以看做一类

##常用模式
####推
什么是推模式？推模式就是，用户A关注了用户B，用户B每发送一个动态，后台遍历用户B的粉丝，往他们粉丝的feed里面推送一条动态。 
####拉
与推模式相反，拉模式则是，用户每次刷新feed第一页，都去遍历关注的人，把最新的动态拉取回来。
####推拉结合
这是一种折中的解决方案，就是在线推，离线拉。

###拉模式
一开始，我们使用的是拉模式pull。
怎么拉呢
1、取用户关系
【我的好友、我关注的 =》并集去重】、我封锁的 =》 差集
2、遍历好友动态，写入timeline,并且记录下pullTime
代码如下：
```
		$follower = new follower($userId);
		$blocking = new blocking($userId);
		$blocked  = new blocked($userId);
		$marker   = new marker($userId);
		$timeline = new timelineModel($userId);

		$pullTime  = $marker->select('pull');
		$followers = $follower->page(0, -1);
		$blockings = $blocking->page(0, -1);
		$blockeds  = $blocked->page(0, -1);

		$followers[$userId] = 0;
		
		foreach($followers as $followerId=>$time) {
			if(isset($blockings[$followerId]) or isset($blockeds[$followerId])) {
				continue;
			}

			$followerMarker = new marker($followerId);
			$followerPush   = $followerMarker->select('push');
			$followerMarker = null;
			if($pullTime>=$followerPush) {
				continue;
			}

			$followerPost = new post($followerId);
			$postResults  = $followerPost->range($pullTime);
			foreach($postResults as $postId=>$time) {
				$counter = new counter($postId);
				$counter->plus('expo');
				$counter = null;
				$timeline->insert($postId, $time);
			}
		}

		$marker->reset('pull', time());

		return $timeline->range($pullTime);
```

这样会出现一个什么问题呢？
我们来看一个细节

![输入图片说明](https://static.oschina.net/uploads/img/201608/04115047_OvaK.png "在这里输入图片标题")
在同一秒内，写入的操作和读的操作会引起数据丢失或者数据重叠的情况。
```
if($pullTime>=$followerPush) {
    continue;
}

```
怎么解决这个地方呢
我们将时间进行了微秒操作，就是先获得时间的微秒时间戳，然后加上一个三位数的random，支持，在同一个时间操作同步的情况，应该是极限接近于0。
```
$microTime = microtime(true);
		$microTime = intval(($microTime-intval($microTime)) * self::CARDINAL_NUMBER);
		$time = $time * self::CARDINAL_NUMBER + $microTime;
		return bcadd($time, rand(1,999) / 1000 ,3);
```

至此，第一版已经完成了。完全采用pull模式。由用户的pullTime去确定是否需要更新timeline。

###存在问题
第一版完成后有哪些问题呢：
1.用户关系更改，timeline无法及时更迭
2.实时性不强