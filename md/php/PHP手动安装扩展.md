CreateTime:2015-11-23 15:06:09.0

#### 进入源码目录下的EXT

    cd php/ext/*
    
#### 使用PHP安装扩展工具phpize
    
    /usr/local/php/bin/phpize

#### 进入源码目录编译安装
    
可以看到再EXT目录下生成里一些文件，执行

    ./configure --enable-* --with-php-config=/usr/local/php/bin/php-config
    make
    make install

#### 修改PHP.ini

    extension = *.so