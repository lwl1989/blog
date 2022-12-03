CreateTime:2020-03-03 12:04:23.0

先描述下db结构吧

order表有
	orderId userId status goodsId
wallet表有
	userId residue status expire


## 复盘逻辑

### 最早时间的逻辑
之前的逻辑是：

```
select residue from wallet
fee = 购买值
foreach wallet as w {
   if w.residue > fee {
       fee = 0
        update wallet set residue = (w.residue - fee) where id = w.id
   }else{
        fee = fee - w.residue
		 update wallet set residue = 0 where id = w.id
   }
   if fee == 0 {
       break
   }
}
if fee == 0 {
   update order set status = 1 where id = (orderId)
}
```

因为系统一直没啥并发，所以之前一直无需考虑幻读之类的问题

### 改版1逻辑

直到前几天，同一时间收到了同一个用户的请求？？
同一秒同一个用户不知道怎么操作的，发起了2个请求。

ok，同个商品购买的两次（商品锁现在也加了）

但是扣费只扣一遍

为啥呢？

```
  xxx := w.residue - fee
  update wallet set residue = xxx where id = w.id
```

。。。 好吧  2个sql同样的执行

于是我改成依赖sql的方式实现了语句

```
var err error
defer func() {
    if err == nil {
		o.commit()
		//send kafka
	}else{
		o.rowback()
	}
}
res,err = o.exec(update wallet set residue = residue - ? where id = ? and residue >= ?)
if err != nil {
    return false
}
if res.affectd == 0 {
	return false
}

//update order status
return true
```

不知道看官发现了BUG没。

## 总结

defer内的条件一定要总结啊！！！！

```
if res.affectd == 0 {
    err = errors.New("buy err: sql affected rows 0")
	return false
}
```
