CreateTime:2015-10-08 10:33:04.0


###1.hiveserver2启动后，beeline不能连接的涉及的问题：
原因：权限问题
解决：
```
/user/hive/warehouse
/tmp
```
/history (如果配置了jobserver 那么/history也需要调整)
这三个目录，hive在运行时要读取写入目录里的内容，所以把权限放开，设置权限：
```
hadoop fs -chmod -R 777 /tmp
hadoop fs -chmod -R 777 /user/hive/warehouse
```
###2.beeline 链接拒绝报错信息
原因：官方的一个bug
解决：
```
hive.server2.long.polling.timeout
hive.server2.thrift.bind.host //注意把host改成自己的host
```
###3.字符集问题、乱码的、显示字符长度问题的
原因：字符集的问题，乱码问题
解决：hive-site.xml中配置的mysql数据库中去
```
 alter database hive character set latin1;
```
###4.message:For direct MetaStore DB connections
这个是由于我的mysql不再本地(默认使用本地数据库)，这里需要配置远端元数据服务器
hive.metastore.uris
```
thrift://lza01:9083
```
Thrift URI for the remote metastore. Used by metastore client to connect to rem
ote metastore. 然后在hive服务端启动元数据存储服务 hive –service metastore
###5. An exception was thrown while adding/validating class(es) : Specified key was too long; max key length is 767 bytes
修改mysql的字符集
```
alter database hive character set latain1;
```