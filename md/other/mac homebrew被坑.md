CreateTime:2018-08-09 10:11:00.0

<p>由于<code>Homebrew/php</code>自来水<a href="https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FHomebrew%2Fhomebrew-php%2Fissues%2F4721" target="_blank" rel="nofollow">在2018年3月底</a>被<a href="https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FHomebrew%2Fhomebrew-php%2Fissues%2F4721" target="_blank" rel="nofollow">弃用</a>，并将所有PHP公式转移到<code>Homebrew/core</code>，旧的<code>brew tap homebrew/dupes、brew tap homebrew/versions、brew tap homebrew/homebrew-php</code>都会报以下错误（目前国内百度都找不到解决这个问题的方法）：</p> 
<pre><code>Warning: homebrew/dupes was deprecated. This tap is now empty as all its formulae were migrated.
</code></pre> 
<p>安装mac+nginx+mysql可看之前的<a href="https://www.jianshu.com/p/1188425fded8" target="_blank" rel="nofollow">文章</a>,php安装和安装扩展,如memcached、redis等则不可在用<code>brew search php</code>查看扩展<code>brew install php70-memcached</code>这种方式安装扩展了。</p> 
<p><strong>安装php</strong></p> 
<pre><code>brew install php56
</code></pre> 
<p><strong>安装php扩展（难点）</strong><br> 不推荐用 pecl 的方式安装 PHP 扩展。以 php-redis 扩展为例，下载源码包来进行安装：</p> 
<pre><code>wget https://pecl.php.net/get/redis-3.1.3.tgz # 下载源码包
tar -zxvf redis-3.1.3.tgz # 解压
cd redis-3.1.3
phpize # 生成编译配置                 
./configure # 编译配置检测
make # 编译
make install # 安装
</code></pre> 
<p>注意：看清你安装的路径<br> 安装memcach、memcached(<code>php5.几版本的需要安装[memcached-2.2.0.tgz](http://pecl.php.net/get/memcached-2.2.0.tgz) 才不会报错</code>)、mongo、mongodb可去<a href="https://link.jianshu.com?t=http%3A%2F%2Fpecl.php.net%2Fpackage%2Fmongo" target="_blank" rel="nofollow">pecl</a>搜搜下载对应的包。下载下来直接解压安装包，建议把安装包放到php@5.6同一级目录中（如：<code>/usr/local/Cellar</code>）解压了文件就可以跳过之前安装的两步，直接 <code>cd …进入生成编辑配置</code>。</p> 
<p><strong>安装遇到的错误</strong></p> 
<pre><code>1.错误configure：error：请重新安装pkg-config分配
报错信息：Checking for pkg-config… /usr/bin/pkg-config
configure: error: Cannot find OpenSSL’s &lt;evp.h&gt;
解决办法：
wget http://pkgconfig.freedesktop.org/releases/pkg-config-0.28.tar.gz
tar -zxvf pkg-config-0.28.tar.gz 
cd pkg-config-0.28
./configure --with-internal-glib
make &amp;&amp; make install
2.报错信息：fatal error: 'openssl/sha.h' file not found on installation
解决办法：
$ cd /usr/local/include 
$ ln -s ../opt/openssl/include/openssl .
</code></pre> 
<p>&nbsp;</p> 
<p>&nbsp;</p>