---
title: Hive语法手册
date: 2016/04/06 00:00:00
toc: true
tags: 
- hive
categories: 
- hive
---

# 数据操作语句

## DML：装载，插入，更新，删除

## 导入/导出

## 数据检索：查询

### select

#### select语法

```
[WITH CommonTableExpression (, CommonTableExpression)*]    (Note: Only available starting with Hive 0.13.0)
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
  FROM table_reference
  [WHERE where_condition]
  [GROUP BY col_list]
  [ORDER BY col_list]
  [CLUSTER BY col_list
    | [DISTRIBUTE BY col_list] [SORT BY col_list]
  ]
 [LIMIT number]
```

* select语句可以是联合查询(union)的一部分或另一个查询的子查询。
* table_reference表示实际的查询对象，可以是一个普通表，一个视图，一个join或一个子查询。
* 表名和列名不区分大小写。
	* 在Hive 0.12和更早的版本中，表和列名中只允许字母数字和下划线。
	* 在Hive 0.13及更高版本中，列名可以包含任何Unicode字符（见[HIVE-6013](https://issues.apache.org/jira/browse/HIVE-6013)），反引号（\`\`）可指定的任何列名。可以使用双反引号（\`\`）来表示一个反引号字符。
	* 要恢复到0.13.0版本以前的行为，并限制列的名称字母，数字和下划线字符，设置属性`hive.support.quoted.identifiers`为none。在此配置中，反引号(\`\`)中的字符被解释为正则表达式。有关详细信息，请参阅[Supporting Quoted Identifiers in Column Names](https://issues.apache.org/jira/browse/HIVE-6013）。另见[正则表达式](#####正则表达式)。
* 简单查询。例如，下面的查询检索所有列，并从表T1的所有行。

```
SELECT * FROM t1
```

```
注意使用0.13及以上版本，FROM是可选的，如select 1 + 1
```

* 获取当前数据库（0.13及以上版本），使用current_database（）函数：

```
SELECT current_database()
```

* 指定数据库，在表名前指定数据库名（"db_name.table_name"Hive 0.7开始）或在查询语句之前使用`use`语句。
“db_name.table_name”允许访问不同数据库的表。
use语句对之后所有的HiveQL语句生效，使用default重置为默认数据库。

```
USE database_name;
SELECT query_specifications;
USE default;
```

##### where子句

WHERE条件是一个布尔表达式。例如，下面的查询只返回美国区域数量大于10的销售记录。Hive的WHERE子句中支持许多[操作符和UDF](####操作符和用户自定义函数(UDFS))。

```
SELECT * FROM sales WHERE amount > 10 AND region = "US"
```

0.13以上版本也支持在where子句中使用一些子查询。

##### all和distinct子句

ALL和DISTINCT选项指定重复的行是否应该返回。如果没有这些选项给出，默认是所有（返回所有匹配的行）。DISTINCT指定结果集中删除重复的行。注意：Hive从1.1.0版本开始支持`SELECT DISTINCT *`语句。[HIVE-9194](https://issues.apache.org/jira/browse/HIVE-9194)

```
hive> SELECT col1, col2 FROM t1
    1 3
    1 3
    1 4
    2 5
hive> SELECT DISTINCT col1, col2 FROM t1
    1 3
    1 4
    2 5
hive> SELECT DISTINCT col1 FROM t1
    1
    2
```

ALL和DISTINCT也可以在UNION子句中使用-见[Union](####Union)以获取更多信息。

##### 基于分区（partition）的查询
一般情况下，一个SELECT查询扫描整个表。如果一个表使用PARTITIONED BY子句创建，查询语句可以仅查询指定分区表的一小部分。Hive在where子句或Join子句中进行分区修剪，例如：表page_views使用列date进行分区，以下查询语句仅检索2008-03-01和2008-03-31之前的行。

```
SELECT page_views.*
FROM page_views
WHERE page_views.date >= '2008-03-01' AND page_views.date <= '2008-03-31'
```

如果表page_views与表dim_users关联，可以像下面一样在on子句中指定分区范围。

```
SELECT page_views.*
FROM page_views JOIN dim_users
  ON (page_views.user_id = dim_users.id AND page_views.date >= '2008-03-01' AND page_views.date <= '2008-03-31')
```

另见[Group By](####Group By)。

另见[Sort By / Cluster By / Distribute By / Order By](#### Sort/Distribute/Cluster/Order By)

##### having子句

Hive 0.7.0版本增加了对HAVING子句的支持。在旧版本中可以使用子查询达到同样的效果：

```
SELECT col1 FROM t1 GROUP BY col1 HAVING SUM(col2) > 10
```

```
SELECT col1 FROM (SELECT col1, SUM(col2) AS col2sum FROM t1 GROUP BY col1) t2 WHERE t2.col2sum > 10
```

##### limit子句

限制指示要返回的行数。返回的行被随机选择。下面的查询从表T中随机返回1到5行。

```
SELECT * FROM t1 LIMIT 5
```

* Top k查询，下面的语句返回Top 5的销售记录。

```
SET mapred.reduce.tasks = 1
SELECT * FROM sales SORT BY amount DESC LIMIT 5
```

`这个列子应该不对`

##### 正则表达式

0.13.0及更高版本中，如果配置了`hive.support.quoted.identifiers`为`none`，则在select语句支持查询基于正则表达式的列名。

* 我们使用Java正则表达式的语法。尝试[http://www.fileformat.info/tool/regex.htm](http://www.fileformat.info/tool/regex.htm)用于测试。
* 下面的查询将选择所有ds或hr的列。

```
SELECT `(ds|hr)?+.+` FROM sales
```

#### Group By

##### Group By语法

```
groupByClause: GROUP BY groupByExpression (, groupByExpression)*
 
groupByExpression: expression
 
groupByQuery: SELECT expression (, expression)* FROM src groupByClause?
```

在group by表达式中，列由名称指定，而不是列编号，在Hive 0.11及更高版本中，列可以通过位置指定。需要配置`hive.groupby.orderby.position.alias`为`true`（默认为`false`）。

##### 简单例子

为了计算表中的行数：

```
SELECT COUNT(*) FROM table2;
```
注意在不包含[HIVE-287](https://issues.apache.org/jira/browse/HIVE-287)的Hive版本中，需要使用`COUNT(1)`代替`COUNT(*)`

为了计算不同性别的用户数量，可以使用以下查询。

```
INSERT OVERWRITE TABLE pv_gender_sum
SELECT pv_users.gender, count (DISTINCT pv_users.userid)
FROM pv_users
GROUP BY pv_users.gender;
```

多个聚合可以在同一时间内完成，但是，没有任何两个聚合可以使用不同的列。例如，以下是可能的，因为count(DISTINCT)和sum(DISTINCT)指定了相同的列：

```
INSERT OVERWRITE TABLE pv_gender_agg
SELECT pv_users.gender, count(DISTINCT pv_users.userid), count(*), sum(DISTINCT pv_users.userid)
FROM pv_users
GROUP BY pv_users.gender;
```

注意在不包含[HIVE-287](https://issues.apache.org/jira/browse/HIVE-287)的Hive版本中，需要使用`COUNT(1)`代替`COUNT(*)`

如下查询是不允许的，在同一个查询中不允许使用多个DISTINCT表达式。

```
INSERT OVERWRITE TABLE pv_gender_agg
SELECT pv_users.gender, count(DISTINCT pv_users.userid), count(DISTINCT pv_users.ip)
FROM pv_users
GROUP BY pv_users.gender;
```

##### Select语句和group by子句

当使用group by子句，select语句只能包含列包括在GROUP BY子句。当然，有很多聚合功能（例如count）同样可以用在select子句中。
让我们以一个简单的例子：

```
CREATE TABLE t1(a INTEGER, b INTGER);
```
一个使用上表的group by查询可能是这样的：

```
SELECT
   a,
   sum(b)
FROM
   t1
GROUP BY
   a;
```
上面的查询可以正常运行，因为select子句包括a(使用列a做group by)和一个聚合函数(sum(b))。
但是，下面的语句不能运行：
```
SELECT
   a,
   b
FROM
   t1
GROUP BY
   a;
```
因为select子句中有一个未包含在group by子句中的附加列(列 b)，也不是一个聚合函数。这是因为，如果表是这样的：

a|b
-|-
100|1
100|2
100|3
由于仅在a上进行了组合，那么在组a=100中，b应该显示为什么值呢？人们可以说，它应该是第一个值或最​低值，但我们都同意，有多种可能的选择。Hive抛弃了这种猜测，认为在select子句中拥有group by子句中不存在的列是无效的SQL(准确地说，是HQL)。

##### 高级功能

###### 多组插入
聚合或简单查询的结果可以输出到多个表甚至是hdfs(使用hdfs操作)上，例如，在按性别分类的同时，有一个需求是按年龄分类。一个可行的查询是这样的：

```
FROM pv_users
INSERT OVERWRITE TABLE pv_gender_sum
  SELECT pv_users.gender, count(DISTINCT pv_users.userid)
  GROUP BY pv_users.gender
INSERT OVERWRITE DIRECTORY '/user/facebook/tmp/pv_age_sum'
  SELECT pv_users.age, count(DISTINCT pv_users.userid)
  GROUP BY pv_users.age;
```

###### map端聚合

hive.map.aggr控制我们如何做聚合，默认为false，如果它被设置为true，Hive将在map任务直接做的第一级集合。
这通常提供了更好的效率，但可能需要更多的内存来运行工作。

```
set hive.map.aggr=true;
SELECT COUNT(*) FROM table2;
```

注意在不包含[HIVE-287](https://issues.apache.org/jira/browse/HIVE-287)的Hive版本中，需要使用`COUNT(1)`代替`COUNT(*)`

###### Grouping Sets, Cubes, Rollups, 和GROUPING__ID函数

```
版本：
rouping sets, CUBE and ROLLUP操作符,和GROUPING__ID函数在Hive 0.10.0版本中加入。
```

见[增强聚合，多维数据集，分组和汇总](###增强聚合、多维数据集、分组和汇总)获取有关这些聚合操作的详细信息。

另见JIRAs：
[HIVE-2397](https://issues.apache.org/jira/browse/HIVE-2397)group by支持rollup选项
[HIVE-3433](https://issues.apache.org/jira/browse/HIVE-3433)实现CUBE和ROLLUP操作符
[HIVE-3471](https://issues.apache.org/jira/browse/HIVE-3471)实现Hive sets分组集
[HIVE-3613](https://issues.apache.org/jira/browse/HIVE-3613)实现GROUPING_ID函数

#### Sort/Distribute/Cluster/Order By

描述select子句中的ORDER BY, SORT BY, CLUSTER BY, 和DISTRIBUTE BY的语法。

##### Order By语法

Hive QL中的Order By语法跟SQL语言中的ORDER BY类似。

```
colOrder: ( ASC | DESC )
colNullOrder: (NULLS FIRST | NULLS LAST)           -- (Note: Available in Hive 2.1.0 and later)
orderBy: ORDER BY colName colOrder? colNullOrder? (',' colName colOrder? colNullOrder?)*
query: SELECT expression (',' expression)* FROM src orderBy
```

"order by"子句具有很多局限性，在严格模式(设置了`hive.mapred.mode=strict`)中，order by子句后需要跟上`limit`子句，若将`hive.mapred.mode`设置为`nonstrict`则不需要`limit`子句。原因是，为了将所有的记录排序，只能使用一个reducer，如果数据量太大，单个reducer将花费很长的时间才能执行完成。

需要注意的是列由列名指定，则不是位置编号，可以在0.11.0及更新版本中使用配置`hive.groupby.orderby.position.alias`为`true`（默认为false）来启用位置编号。
默认的排序顺序是升序（ASC）。
在Hive 2.1.0及更高版本中，支持空排序，默认空排序顺序为ASC顺序是NULLS FIRST，而默认为空的倒序排列顺序是NULLS LAST。

##### Sort By语法

Sort By语法跟SQL语言中的ORDER BY类似。

```
colOrder: ( ASC | DESC )
sortBy: SORT BY colName colOrder? (',' colName colOrder?)*
query: SELECT expression (',' expression)* FROM src sortBy
```
Hive在将数据传递到reducer之前，使用sort by对行进行排序，排列顺序取决于列的类型，如果列是numeric类型，则按数值顺序排序，如果是string类型，则按字典顺序排序。

##### Sort By和Order By的区别

"order by"和"sort by"的区别在于，前者对所有输出进行排序，后者仅对一个reducer中的输出进行排序，如果存在多个reducer，sort by只能进行部分排序。

注意：可能会对Sort By和CLUSTER BY的区别感到困惑，CLUSTER BY在有多个reducer的情况下，为了将数据均衡地分布数据，将数据按字段进行分区并排序。

基本上，在每个reducer中的数据将会按照用户指定的字段进行排序。如下示例：

```
SELECT key, value FROM src SORT BY key ASC, value DESC
```
这个查询可能有2个reducer，输出分别是：

```
0   5
0   3
3   6
9   1
```
```
0   4
0   3
1   1
2   5
```

在一次transform后，变量将会被认为是string类型，意味着数值型将按照字典序进行排序，为了克服这种情况，在使用sort by之前需要使用一个带有cast的子查询，如下：

```
FROM (FROM (FROM src
            SELECT TRANSFORM(value)
            USING 'mapper'
            AS value, count) mapped
      SELECT cast(value as double) AS value, cast(count as int) AS count
      SORT BY value, count) sorted
SELECT TRANSFORM(value, count)
USING 'reducer'
AS whatever
```

##### Cluster By和Distribute By语法

Cluster By和Distribute By主要跟[Transform/Map-Reduce脚本](####Transform和map-reduce脚本)一起使用，但是有时候为了给后续的子查询分区并排序时，这是非常有用的。
Cluster By可以看作是Distribute By和Sort By的合集。
Hive使用Distribute By指定的列将数据分发到reducer中，相同的值将会放到同一个reducer中，但是Distribute By不保证clustering或sorting属性。

例如，我们Distribute By X以下5行记录到2个reducer中：

```
x1
x2
x4
x3
x1
```
reducer 1

```
x1
x2
x1
```

reducer 2

```
x4
x3
```
X相同的值x1分发到reducer1 中，但不能保存其在相邻位置。

如果我们使用Cluster By x，这两个reducer将会进行进一步排序。
reducer 1

```
x1
x1
x2
```

reducer 2

```
x3
x4
```

可以使用Distribute By和Sort By来代替Cluster By，而且分区列和排序列可以是不同的列。

```
SELECT col1, col2 FROM t1 CLUSTER BY col1
```

```
SELECT col1, col2 FROM t1 DISTRIBUTE BY col1
SELECT col1, col2 FROM t1 DISTRIBUTE BY col1 SORT BY col1 ASC, col2 DESC
```

```
FROM (
  FROM pv_users
  MAP ( pv_users.userid, pv_users.date )
  USING 'map_script'
  AS c1, c2, c3
  DISTRIBUTE BY c2
  SORT BY c2, c1) map_output
INSERT OVERWRITE TABLE pv_users_reduced
  REDUCE ( map_output.c1, map_output.c2, map_output.c3 )
  USING 'reduce_script'
  AS date, count;
```

#### Transform和map-reduce脚本

用户可以指定自定义的mapper和reducer，例如，为了执行自定义的mapper-map_script和自定义的reducer-reducer_script，可以使用如下带有TRANSFORM子句的命令嵌入map和reduce脚本。

默认情况下，在传输给自定义脚本时，列将会转换成string类型，并且以tab作为分隔符，同样的，为了区分空字符串，所有的NULL值将被转换成\N字符。自定义脚本的标准输出是tab分隔的string列，任何包括\N的值将被认为是NULL, string类型的结果将转换成表定义中的列类型。用户自定义脚本可以输出debug信息到标准错误，这将在Hadoop任务详细信息页面上显示。这些默认值可以被ROW FORMAT覆盖。

在windows下，使用"cmd /c your_script"。

```
警告
在transformation之前处理任意string列是你的责任，如果有string列包含tab，需要使用[REGEXP_REPLACE](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF#LanguageManualUDF-StringFunctions)并替换tab为其他字符。
```

```
警告
从形式上，MAP...和REDUCE...是SELECT TRANSFORM ( ... )的语法。换句话说，它们作为阅读查询语句的注释或说明。注意：使用这些关键字是危险的，如，输出"REDUCE" 不能强制转换成reduce，输出"MAP"不能强制转换成map。
```
另请参阅[ Sort By / Cluster By / Distribute By ](#### Sort/Distribute/Cluster/Order By)和[Larry Ogrodnek's的博客](http://dev.bizo.com/2009/10/hive-map-reduce-in-java.html)

```
clusterBy: CLUSTER BY colName (',' colName)*
distributeBy: DISTRIBUTE BY colName (',' colName)*
sortBy: SORT BY colName (ASC | DESC)? (',' colName (ASC | DESC)?)*
 
rowFormat
  : ROW FORMAT
    (DELIMITED [FIELDS TERMINATED BY char]
               [COLLECTION ITEMS TERMINATED BY char]
               [MAP KEYS TERMINATED BY char]
               [ESCAPED BY char]
               [LINES SEPARATED BY char]
     |
     SERDE serde_name [WITH SERDEPROPERTIES
                            property_name=property_value,
                            property_name=property_value, ...])
 
outRowFormat : rowFormat
inRowFormat : rowFormat
outRecordReader : RECORDREADER className
 
query:
  FROM (
    FROM src
    MAP expression (',' expression)*
    (inRowFormat)?
    USING 'my_map_script'
    ( AS colName (',' colName)* )?
    (outRowFormat)? (outRecordReader)?
    ( clusterBy? | distributeBy? sortBy? ) src_alias
  )
  REDUCE expression (',' expression)*
    (inRowFormat)?
    USING 'my_reduce_script'
    ( AS colName (',' colName)* )?
    (outRowFormat)? (outRecordReader)?
 
  FROM (
    FROM src
    SELECT TRANSFORM '(' expression (',' expression)* ')'
    (inRowFormat)?
    USING 'my_map_script'
    ( AS colName (',' colName)* )?
    (outRowFormat)? (outRecordReader)?
    ( clusterBy? | distributeBy? sortBy? ) src_alias
  )
  SELECT TRANSFORM '(' expression (',' expression)* ')'
    (inRowFormat)?
    USING 'my_reduce_script'
    ( AS colName (',' colName)* )?
    (outRowFormat)? (outRecordReader)?
```

标准SQL不支持TRANSFORM，TRANSFORM子句在Hive 0.13.0及[HIVE-6415](https://issues.apache.org/jira/browse/HIVE-6415)的后续版本中，通过配置[SQL standard based authorization](https://cwiki.apache.org/confluence/display/Hive/SQL+Standard+Based+Hive+Authorization)禁用。

TRANSFORM示例：
例1：

```
FROM (
  FROM pv_users
  MAP pv_users.userid, pv_users.date
  USING 'map_script'
  AS dt, uid
  CLUSTER BY dt) map_output
INSERT OVERWRITE TABLE pv_users_reduced
  REDUCE map_output.dt, map_output.uid
  USING 'reduce_script'
  AS date, count;
FROM (
  FROM pv_users
  SELECT TRANSFORM(pv_users.userid, pv_users.date)
  USING 'map_script'
  AS dt, uid
  CLUSTER BY dt) map_output
INSERT OVERWRITE TABLE pv_users_reduced
  SELECT TRANSFORM(map_output.dt, map_output.uid)
  USING 'reduce_script'
  AS date, count;
```
例2:

```
FROM (
  FROM src
  SELECT TRANSFORM(src.key, src.value) ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.TypedBytesSerDe'
  USING '/bin/cat'
  AS (tkey, tvalue) ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.TypedBytesSerDe'
  RECORDREADER 'org.apache.hadoop.hive.contrib.util.typedbytes.TypedBytesRecordReader'
) tmap
INSERT OVERWRITE TABLE dest1 SELECT tkey, tvalue
```

##### 缺少schema的map-reduce脚本

如果在USING my_script之后没有AS子句，Hive假定脚本的输出包括两个部分，key是第一个tab之前的部分，值是第一个tab之后的部分。注意这与指定AS key, value不同，因为在那种情况下，如果存在多个tab，value将仅包含第一个tab第二个tab之间的部分。

注意我们可以直接使用CLUSTER BY key，而不在脚本中指定输出格式。

```
FROM (
  FROM pv_users
  MAP pv_users.userid, pv_users.date
  USING 'map_script'
  CLUSTER BY key) map_output
INSERT OVERWRITE TABLE pv_users_reduced
  REDUCE map_output.key, map_output.value
  USING 'reduce_script'
  AS date, count;
```

##### TRANSFORM的输出

脚本的输出字段默认以string进行输入，例如：

```
SELECT TRANSFORM(stuff)
USING 'script'
AS thing1, thing2
```
这将可以被强制转换为

```
SELECT TRANSFORM(stuff)
USING 'script'
AS (thing1 INT, thing2 INT)
```

#### 操作符和用户自定义函数(UDFS)

在Beeline或CLi下，使用如下命令来显示最近的文档：

```
SHOW FUNCTIONS;
DESCRIBE FUNCTION <function_name>;
DESCRIBE FUNCTION EXTENDED <function_name>;
```

```
当UDF嵌套在UDF或function中时存在Bug。
当 `hive.cache.expr.evaluation`设置为`true`（默认）时，UDF嵌套在UDF或function中将会产生不正确的结果。这个bug会影响0.12.0, 0.13.0和0.13.1。0.14.0修复了这个bug([HIVE-7314](https://issues.apache.org/jira/browse/HIVE-7314).
这个问题与UDF的实现的getDisplayString方法有关，在Hive user mailing list中的相关[讨论](http://mail-archives.apache.org/mod_mbox/hive-user/201407.mbox/%3cCAEWg7THU-Pr1Dfv_A8VS3Uz5t3ZyJvL0f-bebg4Zb3hXkK-CGQ@mail.gmail.com%3e)
```

##### 内置操作符

###### 关系运算符

###### 算术运算符

###### 逻辑运算符

###### 复杂类型的运算符

##### 内置函数

###### 

###### 

###### 

###### 

###### 

##### 内置聚合函数

##### 内置生成表函数

##### Grouping and Sorting on

##### UDF内部

##### 创建自定义的UDF

#### XPath专用函数

#### Joins

#### Join优化

#### Union

#### 横向视图

### 子查询

### 采样

### 虚拟列

### 窗口和分析函数

### 增强聚合、多维数据集、分组和汇总

## 程序语言：Hive HPL/SQL

## 解释执行计划

