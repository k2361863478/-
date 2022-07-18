SQL优化的几种方式
一、避免操作多余数据
1、使用where条件语句限制要查询的数据，避免返回多余的行。
2、 尽量避免select *，改使用select 列名，避免返回多余的列。
3、若插入数据过多，考虑批量插入。
4、尽量避免同时修改或删除过多数据。
5、尽量避免向客户端返回大数据量。
二、避免删库跑路
6、删除数据时，一定要加where语句。
三、where查询字句优化
7、避免在where 子句中的 “=” 左边进行内置函数、算术运算或其他表达式运算。
8、避免在 where 子句中使用 != 或 <> 操作符。
9、避免在 where 子句中使用or操作符。
10、where子句中考虑用默认值代替null。
11、不要在where字句中使用not in。
11、合理使用exist & in。
12、谨慎使用distinct关键字。
四、limit查询优化
13、查询一条或者最大/最小一条记录，建议使用limit 1。
14、优化limit分页。
五、like语句优化
15、优化like语句。
六、索引优化
16、查询时应避免全表扫描，首先考虑在where和order by设计的列上建立索引。
17、使用联合索引时，注意索引列的顺序，一般遵循最左匹配原则。
18、索引不要超过6个。
19、删除冗余和无效的索引。
20、尽量使用数字型字段。
21、尽可能的使用 varchar/nvarchar 代替 char/nchar。
22、建议使用自增主键。
网上有很多关于SQL优化的文章，写的很好，但我觉得不够系统，不方便记忆。结合实际使用的一些经验，对SQL优化的功能点进行梳理、总结，方便大家使用。

一、避免操作多余数据
1、使用where条件语句限制要查询的数据，避免返回多余的行。
查询学生张三的成绩的年纪是否为18岁
优化前：

select * from student where age = 18;
1
再从查询到的结果种判断是否包含张三
优化后：

select name from student where age = 18 and name = "张三";
1
2、 尽量避免select *，改使用select 列名，避免返回多余的列。
查询所有18岁的学生姓名
优化前：

select * from student where age = 18;
1
优化后：

select name from student where age = 18;
1
3、若插入数据过多，考虑批量插入。
批量插入学生数据

优化前：

##伪代码
for(User u :list){
	 insert into student(age, name, height, weight) values(#{age}, #{name}, #{height}, #{weight}); 
}
1
2
3
4
优化后：

insert into student(age, name, height, weight) values 
    <foreach collection="list" separator="," index="index" item="item">
      (#{age}, #{name}, #{height}, #{weight})
    </foreach>
1
2
3
4
原因：
批量插入性能好，插入效率更高

4、尽量避免同时修改或删除过多数据。
尽量避免同时修改或删除过多数据，因为会造成cpu利用率过高，从而影响别人对数据库的访问，建议分批操作。

5、尽量避免向客户端返回大数据量。
若数据量过大，应该考虑相应需求是否合理。
大数据量的查询，优先使用分页查询。
仍不能满足需求的，考虑使用es 或者 分库分表。
二、避免删库跑路
6、删除数据时，一定要加where语句。
这个不多说，删库跑路 必备技能。

三、where查询字句优化
7、避免在where 子句中的 “=” 左边进行内置函数、算术运算或其他表达式运算。
优化前：

select * from student where Date_ADD(updated_time,Interval 7 DAY) >=now();
select * from student where age + 1 = 18;
select * from student where substring(name,1,1) = `z`;
1
2
3
优化后：

select * from student where updated_time >= Date_ADD(NOW(),INTERVAL - 7 DAY);
select * from student where age = 18 - 1;
select * from student where name like = "z%";
1
2
3
原因：
将导致系统放弃使用索引而进行全表扫描。

8、避免在 where 子句中使用 != 或 <> 操作符。
查询年龄不是18岁的学生

优化前：

select * from student where age != 18;
select * from student where age <> 18;
1
2
优化后：

select * from student where age < 18 union all select * from student where age > 18;
1
原因：
将导致系统放弃使用索引而进行全表扫描。

9、避免在 where 子句中使用or操作符。
查询年龄为17和18岁的学生信息

优化前：

select * from student where age = 18 or age = 17;
1
优化后：

select * from student where age = 18 union all select * from student where age = 17;
1
原因：
将导致系统放弃使用索引而进行全表扫描。

10、where子句中考虑用默认值代替null。
查询未填写城市的学生

优化前：

select * from student where city is null;
1
优化后：

select * from student where city = "0";
1
原因：
不用is null 或者 is not null 不一定不走索引了，这个跟mysql版本以及查询成本有关。把null值，换成默认值，很多时候让走索引成为可能。

11、不要在where字句中使用not in。
查询年龄不是17岁的学生信息

优化前：

select * from student where age not in (17);
1
优化后：

select * from student a left join (select * from student where age = 17 ) b on a.id = b.id where b.id is null;
1
原因：
not in 不走索引，建议使用not exsits 和 left join优化语句。

11、合理使用exist & in。
in是把外表和内表作hash连接，而exists是对外表作loop循环，每次loop循环再对内表进行查询。如果查询的两个表大小相当，那么用in和exists差别不大，则子查询表大的用exists，子查询表小的用in。

12、谨慎使用distinct关键字。
查询所有不重复的用户年龄

优化前：

select distinct * from student 
1
优化后：

select distinct name from student;
1
原因：
当查询很多字段时，如果使用distinct，数据库引擎就会对数据进行比较，过滤掉重复数据，然而这个比较，过滤的过程会占用系统资源，cpu时间。

四、limit查询优化
13、查询一条或者最大/最小一条记录，建议使用limit 1。
查询身高最高的学生

优化前：

select name from student order by height desc;
1
优化后：

select name from student order by height desc limit 1;
1
14、优化limit分页。
年龄从大到小，分页查询学生姓名

优化前：

select name,age from student order by age desc limit 1, 20;
1
优化后：

## #{age}代表上次查询的最小年龄
select name,age from student where age < #{age} order by age desc limit 20;
1
2
注意，此处的age字段应为唯一索引，如果不是唯一索引，会出现数据重复的问题。

原因：
当偏移量最大的时候，查询效率就会越低，因为Mysql并非是跳过偏移量直接去取后面的数据，而是先把偏移量+要取的条数，然后再把前面偏移量这一段的数据抛弃掉再返回的。

五、like语句优化
15、优化like语句。
查询姓名包含张三的学生信息

优化前：

select * from student where name like "%张三%";
1
优化后：

select * from student where name like "张三%";
1
原因：
like语句后跟"%%"走不了索引。

like语句的优化方案，我觉得很有必要新写一篇文章，后面会把链接贴过来。

关于该部分的优化，可以参考另一篇文章链接：
模糊查询优化 like语句（已亲测）

六、索引优化
16、查询时应避免全表扫描，首先考虑在where和order by设计的列上建立索引。
select * from student where age = 18;
select height from student order by height;
1
2
需要为age 和 height 字段加上索引。

17、使用联合索引时，注意索引列的顺序，一般遵循最左匹配原则。
存在索引：

KEY `name_age_height` (`name`, `age`, `height`)
1
存在下面的查询语句：

select * from student where name = "张三";

select * from student where name = "张三" and age = 18;

select * from student where age = 18;
1
2
3
4
5
根据最左匹配原则，上面第三条查询语句，是不会走索引的。

18、索引不要超过6个。
索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。

19、删除冗余和无效的索引。
存在索引：

KEY `height_weight` (`height`, `weight`);
KEY `height` (`height`);
1
2
上面第二个索引属于冗余索引，需要删除掉。

20、尽量使用数字型字段。
若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会 逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

21、尽可能的使用 varchar/nvarchar 代替 char/nchar。
因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

22、建议使用自增主键。
————————————————
版权声明：本文为CSDN博主「终成一个大象」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/Martin_chen2/article/details/121149963
