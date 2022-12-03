CreateTime:2015-10-08 16:04:59.0

<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">hp.ini中有关session的一些设定会影响到session函数的使用，现在以php5版本为例,我们来了解一下php.ini中有关session的设定:</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;处理session存取的模式(预设：files)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.save_handler = files</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session档案存放路径(预设：/tmp)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.save_path = /tmp</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session使用cookie的功能(预设：启动 1)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.use_cookies = 1</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session的名字(预设：PHPSESSID)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.name = PHPSESSID</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;自动启动(预设：关 0，此处可以改为1)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.auto_start = 0</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session使用cookie的生存期，以秒为单位(预设：随浏览器关闭而消失 0)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.cookie_lifetime = 0</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session使用cookie的路径(预设：与domian相同或根路径 /)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.cookie_path = /</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session使用cookie的域名称(预设：空)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.cookie_domain =</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;处理连续资料的方式，本功能只有WDDX模组或PHP内部使用(预设：php)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.serialize_handler = php</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;按千分之一的比率进行垃圾收集</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;垃圾收集的处理几率(预设：1)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.gc_probability = 1</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;设置进程比率，(php5新增参数，预设：1000)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.gc_divisor = 1000</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;(垃圾收集)被处理前的生存期(预设：1440[秒])</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.gc_maxlifetime = 1440</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;PHP 4.2和以前的版本都有个BUG,即使你禁止了”允许注册全局变量”.仍然可以让你在全局变量范围中</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">初始化一个SESSION的值</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;PHP 4.3 和以后的版本会发出相应的警告,你可以禁止警告.PHP5中,只有你打开了bug_compat_42(=ON),</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">警告才会显示.</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.bug_compat_42，0</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.bug_compat_warn = 1</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;session在重新整理时检查session是否还存在(预设：空)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.referer_check =</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;特别设定session值的长度(预设：关)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.entropy_length = 0</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;特别设定session值的文件</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.entropy_file =</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;使用cache限制器(预设：不要cache)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.cache_limiter = nocache</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;使用cache的生存期</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.cache_expire = 180</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;使用sid值(session_id)传送模式(基于安全，预设：关)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.use_trans_sid = 0</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;选择一个HASH函数,0为MD5(128比特强度),1为SHA-1(160比特强度)</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.hash_function = 0</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;定义当转换2进制hash数据为一些可读的数据时,每个字符存储多少个比特.</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;4 比特: 0-9, a-f</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;5 比特: 0-9, a-v</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;6 比特: 0-9, a-z, A-Z, “-”, “,”</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">session.hash_bits_per_character = 5</span><br>
<br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">;URL重指向的标签</span><br>
<span style="line-height:17px;font-family:Verdana, 'BitStream vera Sans', Helvetica, sans-serif;color:#555555;font-size:12px;background-color:#FFFFFF;">url_rewriter.tags = “a=href,area=href,frame=src,input=src,form=fakeentry”</span>