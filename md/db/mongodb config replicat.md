CreateTime:2016-09-30 11:48:52.0

1.WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'
[https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/](https://docs.mongodb.com/manual/tutorial/transparent-huge-pages)


2.mongodb.config  (replica)
```
dbpath=/usr/local/mongodb/data
logpath=/usr/local/mongodb/log/mongodb.log
pidfilepath=/usr/local/mongodb/log/mongodb.pid

port=12345

logappend=true
#fork=true
#journal=true
#oplogSize=2048
#smallfiles=true

replSet=dbSet
```

3.php7  mongodb config
```
$mongodbDsn = 'mongodb://192.168.28.89:12345,192.168.28.89:12346,192.168.28.89:12347';
$options = array(
                        'replicaSet'     => 'dbSet',   //此处replicaSet的值应和上面配置的replSet保持一致
                        'readPreference' => 'primary'
                );
$manager = new  MongoDB\Driver\Manager($mongodbDsn,$options);
$command = new \MongoDB\Driver\Command(['ping' => 1]);
$manager->executeCommand('db', $command);

```