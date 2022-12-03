CreateTime:2019-12-18 18:42:20.0

# 好冷，早知道不写GO了

嗯，就是开个玩笑，冬天有点冷，特别是寒潮来了，各位注意保暖。

# 为什么写这个生成器

最近要写GO项目，然后发现orm着实难用，一个model要去手动写，更坑的是，`号里面的内容，没有自动打印。天好冷吗，手好抖，南方的冬天，你懂的。

像JAVA、PHP等语言，都有成熟的模型生成器，然而Go我并没有找到，可能是我没有和百度（当然还有墙外的哥）达成深度合作吧？为此，懒人李只能造个轮子，为了提高效率(ps:就是想偷懒、摸鱼)。


# 过程分析

那我们要将数据库如何转换成go的代码呢？开始我想的是，直接拿create sql进行解析

比如表cate
```
CREATE TABLE `cate` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL DEFAULT '',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

emm,正则写起来好像有点复杂（ps：好吧，我真不擅长正则，别拆穿，我知道你们有大佬能写出来）

后面遨游各大搜索引擎，终于找到了便捷的方式。

	SELECT COLUMN_NAME,DATA_TYPE,IS_NULLABLE,TABLE_NAME,COLUMN_COMMENT
		FROM information_schema.COLUMNS 
		WHERE table_schema = DATABASE() and table_name='cate'

哇，看结果：
![](https://oscimg.oschina.net/oscnet/up-4c92c7460a08515b0abf33a463559a43805.png)

我想要的东西好像都出来了，神奇动物原来在这里（ps:他还好没在车底,阿杜对不起）

# 结构定义

既然基础结构有了，那不如就直接定义结构把。

```
type column struct {
	ColumnName    string  //字段名
	Type          string   //字段类型
	Nullable      string  //允许为空
	TableName     string  //表名
	ColumnComment string  //字段备注
	Tag           string    //go渲染用的tag
}
```

然后分析下，mysql的类型和go的不一样啊，这时候搞个mapping就好了。

```
var TypeMappingMysqlToGo = map[string]string{
	"int":                "int",
	"double":             "float64",
	"varbinary":          "string",
	// 好多啊
}
```

然后的套路就是内容的生成了

无非是下划线转驼峰，首字符提升大写等等。

```
prefix, ok := convert.TablePrefix[tableRealName]
		if ok {
			tableRealName = tableRealName[len(prefix):]
		}
		tableName := tableRealName

		switch len(tableName) {
		case 0:
			continue
		case 1:
			tableName = strings.ToUpper(tableName[0:1])
		default:
			tableName = strings.ToUpper(tableName[0:1]) + tableName[1:]
		}
		depth := 1
		var content string
		content += "package " + convert.PackageName + "\n\n" //写包名
		content += "type " + tableName + " struct {\n"
		columns, ok := convert.TableColumn[tableRealName]
		for _, v := range columns {
			var comment string
			if v.ColumnComment != "" {
				comment = fmt.Sprintf(" // %s", v.ColumnComment)
			}
			content += fmt.Sprintf("%s%s %s %s%s\n",
				Tab(depth), v.GetGoColumn(prefix, true), v.GetGoType(), v.GetTag("orm"), comment)
		}
		content += Tab(depth-1) + "}\n\n"

		content += fmt.Sprintf("func (%s *%s) %s() string {\n",
			LcFirst(tableName), tableName, "GetTableName")
		content += fmt.Sprintf("%sreturn \"%s\"\n",
			Tab(depth), tableRealName)
		content += "}\n\n"
		convert.writeModel(tableRealName, content) //写文件
```

当当当当，华丽转身

# 使用demo

代码，哟西。https://github.com/go-libraries/model/blob/master/mysql_test.go
```
	Mysql := GetMysqlToGo()
	Mysql.SetDsn(dsn)
	Mysql.SetModelPath("/tmp")
	Mysql.GetTables()
	Mysql.ReadTablesColumns()
	Mysql.SetPackageName("models")
	Mysql.Run()
```
执行结果:

    ll /tmp
    
```
total *
-rw-r--r--  1 limars  wheel  297 12 18 17:59 cate.go
-rw-r--r--  1 limars  wheel  597 12 18 17:59 comment.go
-rw-r--r--  1 limars  wheel  826 12 18 17:59 content.go
......
```

    cat cate.go
```
package models

type Cate struct {
	Id         int    `orm:"id" json:"id"`
	Name       string `orm:"name" json:"name"`
	CreateTime string `orm:"create_time" json:"create_time"`
	UpdateTime string `orm:"update_time" json:"update_time"`
}

func (cate *Cate) GetTableName() string {
	return "cate"
}
```
# 广告广告

诶，像我这么不出名的人，当然是打个人广告了。

给我来个STAR吧，我会写pgsql和Mariadb的。

也会逐步支持bee、gin等框架哒

https://github.com/go-libraries/gmodel

好冷，早知道不写GO了，真的冷。



