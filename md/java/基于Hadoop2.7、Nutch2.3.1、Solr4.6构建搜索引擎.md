CreateTime:2016-05-28 16:50:36.0

### 常规数据处理
    安装要注意的地方：
    1.防火墙
    2.ssh
    3.selinux
    4.JAVA版本（大于1.7，非openjdk）
### 安装Hadoop
core-site.xml
```
        <property>
                <name>io.file.bufer.size</name>
                <value>131072</value>
        </property>

        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://master:9000</value>
        </property>

```
hdfs-site.xml
```
       <property>
                <name>dfs.namenode.name.dir</name>
                <value>/hadoop/namenode</value>
        </property>
        <property>
                <name>dfs.blocksize</name>
                <value>268435456</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>/hadoop/data</value>
        </property>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>master:9001</value>
        </property>

```
yarn-site.xml
```        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property>
                <name>yarn.nodemanager.aus-services.mapreduce.shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>

        <property>
                <name>yarn.resourcemanager.address</name>
                <value>master:8032</value>
        </property>
        <property>
                <name>yarn.resourcemanager.scheduler.address</name>
                <value>master:8030</value>
        </property>
        <property>
                <name>yarn.resourcemanager.resource-tracker.address</name>
                <value>master:8031</value>
        </property>

        <property>
                <name>yarn.resourcemanager.admin.address</name>
                <value>master:8033</value>
        </property>
        <property>
                <name>yarn.resourcemanager.webapp.address</name>
                <value>master:8088</value>
        </property>
```
其他要注意的地方：slvaes文件
                *env.sh的JAVA_HOME
### 安装Hbase
    解压修改配置，注意，最好不使用自身自带zookeeper
```
         <property>
                <name>hbase.rootdir</name>
                <value>hdfs://master:9000/hbase</value>
        </property>
        <property>
                <name>hbase.zookeeper.quorum</name>
                <value>master,slave1,slave2</value>
        </property>
        <property>
                <name>hbase.cluster.distributed</name>
                <value>true</value>
        </property>
        <property>
                <name>hbase.zookeeper.property.dataDir</name>
                <value>/opt/zookeeper/var/data</value>
        </property>
```
其他：JAVA_HOME regionservers
### Nutch安装
####编译
    官网下载2.3.1版本，提示

![输入图片说明](https://static.oschina.net/uploads/img/201605/28164145_pRsB.png "在这里输入图片标题") 

解压，修改ivy/ivy.xml
全局修改版本    

     %s/2.5.2/2.7.2/g  
取消注释

    ：<dependency org="org.apache.gora" name="gora-hbase" rev="0.6.1" conf="*->default" /> 
执行
    ant runtime
### Solr安装 
    解压即可 4.6版本 
记录爬坑：
crawl <seedDir> <crawlID> [<solrUrl>] <numberOfRounds>
直接用 solrurl构建索引会出现奇怪的问题
错误日志里输入 NullpointerException
解决方案：
crawl <seedDir> <crawlID>  <numberOfRounds>
忽略 solrUlr
进行第二步，可以解决
nutch solrindex [<solrUrl>] <crawlID> -reindex  

No IndexWriters activated - check your configuration
解决，修改nuth-site.xml,增加plugin.includes为下述。
protocol-http|urlfilter-regex|parse-(html|tika)|index-(basic|anchor)|indexer-solr|scoring-opic|urlnormalizer-(pass|regex|basic)