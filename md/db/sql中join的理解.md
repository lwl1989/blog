CreateTime:2019-06-04 11:28:46.0

## 前言

为什么会突然写这个blog呢？因为之前有只青蛙小姐姐问我，能不能教她join，当时上大学老师怎么教她也不会。然后本来想面对面交流给她说明，后面阴错阳差，就延误到了现在。

所以我想，我可以提前准备好我想说的东西，记录下来，顺便自己也回忆下join(ps:为什么我需要回忆？因为之前的公司都是面向互联网的、高并发的业务，用join的话，很容易导致数据库出现异常问题，我已经很久没用过了)。

当然有机会的话，我觉得还是要当面给她讲。

## join是什么，为什么要有join

这个就很重要了，学一个东西你都不知道它是什么，有什么应用场景的话，基本是学完就忘。

SQL join 用于根据两个或多个表中的列之间的关系，从这些表中查询数据。

举个🌰：

栗子当然是要用学校的栗子啊，在学校的时候吃栗子还是蛮有感觉的，现在吃个🌰，都习以为常了。
```
学生表（student）      uid,name
成绩表（achievement）  uid,score
```

这个时候我们要怎么去查询学生的成绩呢？

如果不用连表，我们可能就需要2条sql.

	select * from student where uid = x
	select * from achievement where uid = x

这样是不是很繁琐。

假如我们有join了，那就很好操作了。

	select * from student join achievement on student.uid = achievement.uid where uid = x

## join的分类

join分为3种情况，分别是inner、left、right。从字面上，我们就能理解为全、左、右

![](https://oscimg.oschina.net/oscnet/a0b86df6123219538999f7ce473dfeabecd.jpg)

### inner join
inner join(等值连接) 只返回两个表中联结字段相等的行。

怎么理解呢？还是拿学生来举例。

	select * from student inner join achievement on student.uid = achievement.uid where uid = x
	
比如一个班有30个学生，但是只有28个人参加的考试。

如果使用inner会展示出参加考试的学生及其成绩

### left join
left join(左联接) 返回包括左表中的所有记录和右表中联结字段相等的记录

怎么理解呢？还是拿学生来举例。

	select * from student left join achievement on student.uid = achievement.uid where uid = x

比如一个班有30个学生，但是只有28个人参加的考试。

如果使用left会展示出所有的学生，没有成绩的会补充为Null。

### right join
right join(右联接) 返回包括右表中的所有记录和左表中联结字段相等的记录

right join一般用于右表数据比较多的情况。

还是拿学生来举例吧。

	select * from student right join achievement on student.uid = achievement.uid where uid = x

比如一个班去年有33个学生，但是有3个人休学了。如果查询去年的成绩的话。

就会得到33条记录，而休学的3个同学的姓名是无法获知了，会和left join一样补充为null。

### 总结

因此很明显的，可以看出来

inner返回是完全匹配

left和right返回的数据是部分匹配但是会将其中一张表数据匹配到的完全返回，而将不完全的表的值填充为空。

至于left和right表，就这么理解，如果是left,那left左边的表名就是完全返回的，如果是right，则反之。

