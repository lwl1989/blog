CreateTime:2015-09-25 17:02:42.0

###首先添加依赖应用###
```yum install -y gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libpng libpng-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel ncurses curl openssl-devel gdbm-devel db4-devel libXpm-devel libX11-devel gd-devel gmp-devel readline-devel libxslt-devel expat-devel xmlrpc-c xmlrpc-c-devel```


LIBMCRYPT下载地址
```
http://nchc.dl.sourceforge.net/project/mcrypt/Libmcrypt/2.5.8/libmcrypt-2.5.8.tar.gz
./configure
make
make install
```
下载PHP，编译并安装
```
wget http://cn2.php.net/distributions/php-5.6.*.tar.gz
tar zxvf php-5.6.*.tar.gz
cd php-5.6.*

./configure \
--prefix=/usr/local/php \
--with-config-file-path=/usr/local/php/etc \
--with-mysql=/usr/local/mysql \
--with-mysqli=/usr/local/mysql/bin/mysql_config \
--with-mysql-sock=/var/lib/mysql/mysql.sock \
--with-pdo-mysql=/usr/local/mysql \
--with-xpm-dir=/usr/ \
--with-iconv \
--enable-libxml \
--enable-xml \
--enable-bcmath \
--enable-shmop \
--enable-sysvsem \
--enable-inline-optimization \
--enable-opcache \
--enable-mbregex \
--enable-fpm \
--enable-mbstring \
--enable-ftp \
--enable-gd-native-ttf \
--with-openssl \
--enable-pcntl \
--enable-sockets \
--with-xmlrpc \
--enable-zip \
--enable-soap \
--without-pear \
--with-gettext \
--enable-session \
--with-mcrypt \
--with-curl \
--enable-ctype 

make && make install




make && make install

Build complete.
Don't forget to run 'make test'.
```

```
cp /usr/local/php-5.6.*/etc/php-fpm.conf.default php-fpm.conf
//复制一份并重命名


/usr/local/php-5.6.*/sbin/php-fpm
//启动php-fpm
```



修改FPM 配置文件php-fpm.conf
```
pm.max_children = 50
pm.start_servers = 20
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 500
```
去掉分号
```
ln -s /usr/local/php-5.6.*/sbin/php-fpm /bin/php-fpm
cp /usr/local/src/php-5.6.*/php.ini-producsion /usr/local/php-5.6.0/lib/php.ini
```

```
sapi/fpm/init.d-php-fpm /etc/init.d/php-fpm
chkconfig --add php-fpm
service start php-fpm
```

未知模块可以用 ./configure help 进行查看 
##PHP7
```
./configure --prefix=/usr/local/php \
 --with-curl \
 --with-freetype-dir \
 --with-gd \
 --with-gettext \
 --with-iconv-dir \
 --with-kerberos \
 --with-libdir=lib64 \
 --with-libxml-dir \
 --with-mysqli \
 --with-openssl \
 --with-pcre-regex \
 --with-pdo-mysql \
 --with-pdo-sqlite \
 --with-pear \
 --with-png-dir \
 --with-xmlrpc \
 --with-xsl \
 --with-zlib \
 --enable-fpm \
 --enable-bcmath \
 --enable-libxml \
 --enable-inline-optimization \
 --enable-gd-native-ttf \
 --enable-mbregex \
 --enable-mbstring \
 --enable-opcache \
 --enable-pcntl \
 --enable-shmop \
 --enable-soap \
 --enable-sockets \
 --enable-sysvsem \
 --enable-xml \
 --enable-zip
```


配置文件
```
# cp php.ini-development /usr/local/php/lib/php.ini
# cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
# cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
# cp -R ./sapi/fpm/php-fpm /etc/init.d/php-fpm
```