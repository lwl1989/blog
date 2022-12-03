CreateTime:2015-10-13 16:16:37.0

###表语句

create '表名','列族1','列族2',......

disable '表名'    //禁用

drop  '表名'

###数据语句

put '表名','列族:列名','值'......  //添加数据到表  
```
put 'student','t2','num:identify','430221198909227***'
put 'student','t2','num:stu','0221010121*'
put 'student','t1','num:identify','43022119890922****'
put 'student','t1','num:stu','0221010121*'
put 'student','t1','name:real','zhangsan'
put 'student','t1','name:nick','fresh'
```

scan '表名'   //列出表信息
```
scan 'student'
ROW                            COLUMN+CELL                                                                             
 t1                            column=name:nick, timestamp=1444722224618, value=fresh                                  
 t1                            column=name:real, timestamp=1444722221005, value=zhangsan                               
 t1                            column=num:identify, timestamp=1444722427089, value=43022119890922****                  
 t1                            column=num:stu, timestamp=1444722440673, value=0221010121*                              
 t2                            column=num:identify, timestamp=1444722299896, value=430221198909227***                 
 t2                            column=num:stu, timestamp=1444722270125, value=0221010121*                              
2 row(s) in 0.0200 seconds

```

delete  '表名','行','列族','列名:值'
```
delete 'student','t2','num','stu:0221010121*'
```

get '表名','行列值'
####更多get
```
get 't1′, 'r1' 
get 't1′, 'r1', {TIMERANGE => [ts1, ts2]} 
get 't1′, 'r1', {COLUMN => 'c1'} 
get 't1′, 'r1', {COLUMN => ['c1', 'c2', 'c3']} 
get 't1′, 'r1', {COLUMN => 'c1', TIMESTAMP => ts1} 
get 't1′, 'r1', {COLUMN => 'c1', TIMERANGE => [ts1, ts2], VERSIONS => 4} 
get 't1′, 'r1', {COLUMN => 'c1', TIMESTAMP => ts1, VERSIONS => 4} 
get 't1′, 'r1', 'c1' 
get 't1′, 'r1', 'c1', 'c2' 
get 't1′, 'r1', ['c1', 'c2'] 

```

####更多scan

```
Some examples:

scan 'hbase:meta'
scan 'hbase:meta', {COLUMNS => 'info:regioninfo'}
scan 'ns1:t1', {COLUMNS => ['c1', 'c2'], LIMIT => 10, STARTROW => 'xyz'}
scan 't1', {COLUMNS => ['c1', 'c2'], LIMIT => 10, STARTROW => 'xyz'}
scan 't1', {COLUMNS => 'c1', TIMERANGE => [1303668804, 1303668904]}
scan 't1', {REVERSED => true}
scan 't1', {FILTER => "(PrefixFilter ('row2') AND
    (QualifierFilter (>=, 'binary:xyz'))) AND (TimestampsFilter ( 123, 456))"}
scan 't1', {FILTER =>
    org.apache.hadoop.hbase.filter.ColumnPaginationFilter.new(1, 0)}
For setting the Operation Attributes 
scan 't1', { COLUMNS => ['c1', 'c2'], ATTRIBUTES => {'mykey' => 'myvalue'}}
 scan 't1', { COLUMNS => ['c1', 'c2'], AUTHORIZATIONS => ['PRIVATE','SECRET']}
scan 't1', {COLUMNS => ['c1', 'c2'], CACHE_BLOCKS => false}
scan 't1', {RAW => true, VERSIONS => 10}

```


