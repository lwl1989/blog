CreateTime:2020-11-24 10:52:15.0

gorm目前的稳定版是无法进行批量插入（v2.0已列入开发计划），那么我们要如何解决呢？

# 分析结构




### 列名获取
gorm的结构体有固定tag，那么我们就可以取到列名

当然是通过反射来进行了。


举个例子：
	AlbumId    int64  `gorm:"column:album_id;type:bigint(20)" json:"album_id"`

我们可以看到，tag的值是以;为分隔为键值对，然后键值对是以:为分隔符，那么这样，我们可以很轻松的取到列名。
```
func GetColumnNameObj(obj interface{}) []string {
	fieldNum := reflect.TypeOf(obj).NumField()
	fieldType := reflect.TypeOf(obj)
	var fieldsName []string
	for i := 0; i < fieldNum; i++ {
		tagValue := fieldType.Field(i).Tag.Get("gorm")
		for _, name := range strings.Split(tagValue, ";") {
			if strings.Index(name, "column") == -1 {
				continue
			}
			fieldsName = append(fieldsName, strings.Replace(name, "column:", "", 1))
		}
		fieldsName = append(fieldsName, tagValue)
	}

	return fieldsName
}
```

### 值获取

既然我们能通过反射获取列名，那么获取值当然也是easy.

```
func GetColumnValueObj(obj interface{}) []string {
	fieldNum := reflect.TypeOf(obj).NumField()
	fieldType := reflect.TypeOf(obj)
	var fieldsValue []string
	
	for i := 0; i < fieldNum; i++ {
		fName := fieldType.Field(i).Type.Name()
		// 获取字段类型
		if fName == "string" {
			fieldsValue = append(fieldsValue, "string")
		} else if strings.Index(fName, "uint") != -1 {
			fieldsValue = append(fieldsValue, "uint")
		} else if strings.Index(fName, "int") != -1 {
			fieldsValue = append(fieldsValue, "int")
		}
	
	}

	return fieldsValue
}
```

当然，我们需要处理更多的类型，我这里只处理了通用的(u)int64和string。

### sql封装

既然我们都知道了列名，值，那剩下的东西就简单多了。

拼接sql
```
func BuildSql(columns []string, lenValues int, table Tabler) (insertSql string) {
	fieldNames := strings.Join(columns, ",")
	lenColumns := len(columns)
	var builder strings.Builder
	if lenValues%lenColumns == 0 {
		//长度验证
		var i = 0

		for ; i < lenValues/lenColumns; i++ {
			builder.WriteString("(")
			for j := 0; j < lenColumns; j++ {
				if j == lenColumns-1 {
					builder.WriteString("?")
				} else {
					builder.WriteString("?,")
				}
			}
			builder.WriteString("),")
		}
	}
	params := strings.TrimRight(builder.String(), ",") + ";"
	insertSql = fmt.Sprintf("insert into `%s` (%s) values %s", table.TableName(), fieldNames, params)
	return
}
```

### 接下来就只剩下执行了

整体代码如下：
```
func BatchInsertGromStruct(objs []Tabler) {
	if len(objs) == 0 {
		return
	}
	//获取列名
	columns := GetColumnNameObj(objs[0])
	//获取值
	values := GetColumnValueObj(objs)
	//封装sql
	sql := BuildSql(columns, len(values), objs[0])
	//执行
	GetOrm().Exec(sql, values...)
}

```


# 总结

其他ORM也大同小异可以使用此方式


