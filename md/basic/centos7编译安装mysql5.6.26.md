CreateTime:2015-09-25 11:46:18.0

#### 使用yum安装一些依赖

```
yum -y install make gcc-c++ cmake bison-devel  ncurses-devel
yum install -y gcc gcc-c++ kernel-devel
yum install -y readline-devel pcre-devel openssl-devel openssl zlib zlib-devel pcre-devel
```

#### 下载mysql####

```
wget http://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.26.tar.gz
tar -zxvf mysql-5.6.26.tar.gz
cd mysql-5.6.26
```

#### 使用CMAKE 编译mysql ,mysql 5.5以后都由cmake来编译####

```
cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/usr/local/mysql/data \
-DSYSCONFDIR=/etc \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DMYSQL_UNIX_ADDR=/var/lib/mysql/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci

make && make install
```


```
groupadd mysql
useradd -g mysql mysql
```

```
chown -R mysql:mysql /usr/local/mysql
```

#### 安装sql服务

```
scripts/mysql_install_db 
--basedir=/usr/local/mysql 
--datadir=/usr/local/mysql/data --user=mysql

5.7 以上
./mysqld --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --explicit_defaults_for_timestamp  --initialize
```

#### 将mysql服务文件放入启动项位置
```
cp support-files/mysql.server /etc/init.d/mysql
```
#### 删除默认的配置文件
```
rm -rf /etc/my.cnf
```

#### 开机启动+启动
```
chkconfig mysql on
service mysql start
```
#### 将mysql加入到全局变量中
```
ln -s /usr/local/mysql/bin/mysql /usr/bin/
```


###至此，mysql就可以正常使用了###

### 错误记录：

please install the following Perl modules before executing ./mysql_install_d

yum install -y perl-Module-Install.noarch