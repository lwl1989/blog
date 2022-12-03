CreateTime:2019-11-26 16:10:33.0

闲来无事，便记录几个最近遇到的Pdo细节问题，平常都是用orm的。

# 长连接

在历史的Mysql驱动中，都是使用connect和pconnect来区分长短连接，到了pdo之后，改成了参数。

\PDO::ATTR_PERSISTENT
```
 $dbh = new \PDO(MYSQL_DSN, MYSQL_USER, MYSQL_PASS, array(
            \PDO::ATTR_PERSISTENT => true,
            \PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8',
        ));
```

> 如果想使用持久连接，必须在传递给 PDO 构造函数的驱动选项数组中设置 PDO::ATTR_PERSISTENT 。如果是在对象初始化之后用 PDO::setAttribute() 设置此属性，则驱动程序将不会使用持久连接。

### MySQL server has gone away

某个mysql长连接很久没有新的请求发起，达到了server端的timeout，被server强行关闭。此后再通过这个connection发起查询的时候，就会报错server has gone away，这种情况特别容易出现在脚本里。(max_allowed_packet 是包太大造成的连接中断，这里不予以讨论)

解决方案：

1. 修改my.cnf的 wait_timeout、interactive_timeout 

2. 在连接里初始化后执行sql,sql = "set interactive_timeout=秒数";


明明是长连接，但是为什么会产生是去连接呢？

# 错误与错误处理

PDO 提供了三种不同的错误处理模式，以满足不同风格的应用开发

1. PDO::ERRMODE_SILENT
2. PDO::ERRMODE_WARNING
3. PDO::ERRMODE_EXCEPTION

## PDO::ERRMODE_SILENT

此为默认模式。

PDO 将只简单地设置错误码，可使用 PDO::errorCode() 和 PDO::errorInfo() 方法来检查语句和数据库对象。

如果错误是由于对语句对象的调用而产生的，那么可以调用那个对象的 PDOStatement::errorCode() 或 PDOStatement::errorInfo() 方法。

如果错误是由于调用数据库对象而产生的，那么可以在数据库对象上调用上述两个方法。

##  PDO::ERRMODE_WARNING

除设置错误码之外，PDO 还将发出一条传统的 E_WARNING 信息，所以一般用于调试。

## PDO::ERRMODE_EXCEPTION

除设置错误码之外，PDO 还将抛出一个 PDOException 异常类并设置它的属性来反射错误码和错误信息。

异常模式另一个非常有用的是，相比传统 PHP 风格的警告，可以更清晰地构建自己的错误处理，而且比起静默模式和显式地检查每种数据库调用的返回值，异常模式需要的代码/嵌套更少。

demo(异常模式):
```
<?php
$dsn = 'mysql:dbname=testdb;host=127.0.0.1';
$user = 'dbuser';
$password = 'dbpass';

try {
    $dbh = new PDO($dsn, $user, $password);
    $dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    echo 'Connection failed: ' . $e->getMessage();
}
?>
```

默认模式:
```
 try {
                $pdo = Mysql::getInstance();
                $pdo = $pdo->getPdo();

                $sql = sprintf('insert into  '.$this->getTableName().' (`job_number`, `point_data`, `sign_time`) 
                            values(\'%s\',\'%s\',\'%s\')',
                    $this->jobNum, $this->__toString(), date('Y-m-d H:i:s'), date('Y-m-d H:i:s', $this->verifyTimeMillis/1000));
                $sth = $pdo->prepare($sql);
                if ($sth) {
                    if ($sth->execute()) {
                        $this->id = $pdo->lastInsertId();
                        $success = true;
                        break;
                    }
                    $code = $sth->errorCode();
                    if($code == '23000') {
                        $success = true;
                        //["23000",1062,"Duplicate entry '1698-2019-11-26 13:43:57' for key 'job_number_time_unique'"]
                        if($this->id == 0) {
                            $this->checkExists();
                        }
                        break;
                    }
                     if($code == 'HY000') {
                            Mysql::getInstance()->reConnection();
                           continue;
                     }

                     if(!empty($code)) {
                             Logs::writeErrorLog('数据操作失败 db error code:'. $code.'message:'.json_encode($info));
                             continue;
                     }

                    if($code == '42S02') {
                        $this->autoCreateTable();
                    }

                } else {
                    throw new \Exception('insert db error with content:' . $this->__toString());
                }
            } catch (\Exception $exception) {
                $e = $exception;
            }
```



