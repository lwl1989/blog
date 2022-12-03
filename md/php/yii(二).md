CreateTime:2016-03-29 16:54:48.0

##数据库操作
  
常用操作就不多做啰嗦了

下面说几点要注意的

###where  andWhere orWhere
```
$status = 10;
$search = 'yii';

$query->where(['status' => $status]);

if (!empty($search)) {
    $query->andWhere(['like', 'title', $search]);
}
$status = [1,2,3,4,5]
if (!empty($status)) {
    foreach($status as $val){
        $query->orWhere(['status' => $val]);
    }
}
```

###in的几种写法
```
  $status = [1,2,3,4,5]
  
  $query->where(['status'=>$status]);
  $query->where(['in','status',$status]);

  $db->createCommand('select * from A where status in (:statua)',[$status]);//错误的 select * from A where status in (1)


  $status = '1,2,3,4,5';
  $db->createCommand('select * from A where status in (:statua)',[$status]);//依旧错误的  select * from A where status in ('1,2,3,4,5')

 
  //子查询写法
   $query->where(['in','status',new Query()->from()->where()->queryAll()]);
   $query->where(['status'=>new Query()->from()->where()->queryAll()]);
  
```

###join
```
   $query->leftJoin('B','B.id=A.id');//其他同 innerJoin rightJoin
   $query->join('left join','B','B.id=A.id');//其他同 innerJoin rightJoin

```

###最自然的方法
```
    Yii:$app->getDb()->createCommond('mysql语句')->execute();  //or queryOne queryAll
```