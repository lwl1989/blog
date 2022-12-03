CreateTime:2015-11-14 21:27:51.0

### 测试源码：
```
header('Content-type:text/html;charset=utf-8;');
//for($num=0;$num<8;$num++){
    insert();
    insert(false);
//}
function insert($myisam =true){
    $mysqli = new mysqli('127.0.0.1','root','sa','t100w');
    $start = getMillisecond();
    $table = 'users';
    $type = 'innodb_group_h';
    if($myisam){
        $table.='_myisam';
        $type = 'myisam_group_h';
    }
    $sql = ' SELECT count(*),gender FROM `'.$table.'` group by gender having count(*)>10000';
    /*for($i = 0 ;$i<10000;$i++) {
        $data = array(
            'username' => rand_string(),
            'password' => '123456',
            'gender' => rand_gender(),
            'mobile' => $i % 6 == 0 ? rand_number() : '',
            'email' => $i % 7 == 0 ? rand_string(10) : '',
            'actived' => $i % 3 == 0 ? 1 : 0,
            'created' => date('Y-m-d H:i:s', time()),
            'is_del' => 0
        );

        $sql = 'insert into '.$table.'(username,password,gender,mobile,email,actived,created,is_del) VALUES(';
        $j=0;
        foreach ($data as $d) {
            if($j==7)
                $sql .="'$d'";
            else
                $sql .= "'$d'".',';
            $j++;
        }
        $sql .= ')';
        $mysqli->query($sql);
        unset($data);
    }*/
    $mysqli->query($sql);
    $exec_time = (getMillisecond()-$start);
    $time_sql = "insert into time_looks(engine_table,exec_time) VALUES('$type','$exec_time')";
    $mysqli->query($time_sql);
    $mysqli->close();
}
```
### 测试比较（极快表示小于等于1MS）
#### insert比较

myisam:平均5300MS
innodb:平均22300MS


![输入图片说明](https://static.oschina.net/uploads/img/201511/14203123_anAZ.png "在这里输入图片标题")

**可以看出myisam写入速度是innodb的3-4倍**

大小比较：

![输入图片说明](https://static.oschina.net/uploads/img/201511/14203713_UJnZ.png "在这里输入图片标题")

**innodb占据空间近2倍于myisam**

#### select比较
##### 普通where子句
```
sqL:SELECT `id`, `username`, `password`, `gender`, `mobile`, `email`, `actived`, `created`, `is_del` FROM `users` WHERE actived=1 
```
![输入图片说明](https://static.oschina.net/uploads/img/201511/14204416_VOrR.png "在这里输入图片标题")

**myisan:极快
innodb:慢**


```
sqL:SELECT `id`, `username`, `password`, `gender`, `mobile`, `email`, `actived`, `created`, `is_del` FROM `users` WHERE actived=1 and gender=1
```
![输入图片说明](https://static.oschina.net/uploads/img/201511/14204822_J4Ht.png "在这里输入图片标题")

制定越精确的条件，innodb速度提高。


##### like速度比较

![输入图片说明](https://static.oschina.net/uploads/img/201511/14205417_y5ow.png "在这里输入图片标题")

**myisan:极快
innodb:慢**

##### group by 、 distinct
```
 select count(*) from users group by gender;
```
![输入图片说明](https://static.oschina.net/uploads/img/201511/14210510_Rdy2.png "在这里输入图片标题")
innodb稍胜
```
 select distinct(gender) from users where 1;
```

![输入图片说明](https://static.oschina.net/uploads/img/201511/14210917_bSI9.png "在这里输入图片标题")

**myisam稍胜**

##### having
	
```
 SELECT count(*),gender FROM `users` group by gender by count(*)>10000
```

![输入图片说明](https://static.oschina.net/uploads/img/201511/14212609_dkBi.png "在这里输入图片标题")

**myisam近2倍**



GROUP BY,WHERE,HAVING之间的区别和用法
1.WHERE 子句用来筛选 FROM 子句中指定的操作所产生的行。
2.GROUP BY 子句用来分组 WHERE 子句的输出。
3.HAVING 子句用来从分组的结果中筛选行。

#### update比较
![输入图片说明](https://static.oschina.net/uploads/img/201511/14210108_Ryjo.png "在这里输入图片标题")