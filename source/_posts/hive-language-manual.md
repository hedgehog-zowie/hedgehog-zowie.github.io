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

以下操作符比较输入的操作数并且依据操作数之前的比较生成TRUE或FALSE值。

操作符|操作数类型|描述
--|--|--
A = B|所有基本类型|如果表达式A与表达式B相等，则为TRUE，否则为FALSE。
A == B|所有基本类型|同 = 
A <=> B|所有基本类型|若操作数非空（non-none），则返回与=相同的结果；若存存空操作数，当两个都为空操作符时，返回TRUE，仅一个为空时，返回FALSE（截至版本[0.9.0](https://issues.apache.org/jira/browse/HIVE-2810)）；
A <> B|所有基本类型|如果A或B为NULL，返回NULL；如果A不等于B，返回TRUE，否则返回FALSE
A != B|所有基本类型|同 <> 
A < B|所有基本类型|如果A或B为NULL，返回NULL；如果A小于B，返回TRUE，否则返回FALSE
A <= B|所有基本类型|如果A或B为NULL，返回NULL；如果A小于等于B，返回TRUE，否则返回FALSE
A > B|所有基本类型|如果A或B为NULL，返回NULL；如果A大于B，返回TRUE，否则返回FALSE
A >= B|所有基本类型|如果A或B为NULL，返回NULL；如果A大于等于B，返回TRUE，否则返回FALSE
A [NOT] BETWEEN B AND C|所有基本类型|如果A、B或C为NULL，返回NULL；如果A大于等于B并且A小于等于C，返回TRUE，否则返回FALSE。可以使用NOT关键字（（截至版本[0.9.0](https://issues.apache.org/jira/browse/HIVE-2810)。）
A IS NULL|所有类型|如果表达式A计算为NULL，则返回TRUE，否则返回FALSE。
A IS NOT NULL|所有类型|如果表达式A计算不为NULL，则返回TRUE，否则返回FALSE。
A [NOT] LIKE B|strings|如果A或B为NULL，返回NULL；如果字符串A正则匹配表达式B（SQL正则），则返回TRUE，否则返回FALSE。挨个字符进行比较，表达式B中的_字符表示匹配任意字符（类型posix正则表达式的.），%字符表示匹配A中的一串字符（类似posix正则表达式中的.*）。例如：'foobar' like 'foo'为FALSE，则'foobar' like 'foo_\_\_'和'foobar' like 'foo%'为TRUE。
A RLIKE B|strings|如果A或B为NULL，返回NULL；如果A中的任意子字符串正则匹配B（JAVA正则），则返回TRUE，否则返回FALSE。例如：'foobar' RLIKE 'foo'为TRUE，'foobar' RLIKE '^f.\*r$'
A REGEXP B|strings|同 RLIKE

###### 算术运算符

以下操作符支持操作数间常见的运算，返回值均为数值型，如果存在任意值为NULL，则返回值也为NULL。

操作符|操作数类型|描述
-|-|-
A + B|
A - B|
A * B|
A / B|
A % B|
A & B|
A ^ B|
~A|

###### 逻辑运算符

以下操作符支持创建逻辑表达式。所有的返回值均为TRUE，FALSE或NULL，取闷在于操作数的boolean值。NULL标识为"unknown"，因此如果结果取决于"unknown"状态，那么结果本身是"unknown"。

操作符|操作数类型|描述
-|-|-
A AND B|boolean|
A && B|boolean|
A OR B|boolean|
A || B|boolean|
NOT A|boolean|
!A|boolean|
A IN (val1, val2, ...)|boolean|
A NOT IN (val1, val2, ...)|boolean|
[NOT] EXISTS(subquery)||

###### 复杂类型的运算符

以下函数构造复杂对象的实例。

操作符|操作数类型|描述
-|-|-
map|
struct|
named_struct|
array|
create_union|

###### 复杂类型操作

以下函数提供访问复杂对象的机制。

操作符|操作数类型|描述
-|-|-
A[n]|
M[key]|
S.x|

##### 内置函数

###### 数学函数

Hive支持以下内置数学函数，当参数为NULL时大部分返回NULL。

返回类型|名称|描述
-|-|-
DOUBLE|round(DOUBLE a)|
DOUBLE|round(DOUBLE a, INT d)|
DOUBLE|bround(DOUBLE a)|
DOUBLE|bround(DOUBLE a, INT d)|
BIGINT|floor(DOUBLE a)|
BIGINT|ceil(DOUBLE a), ceiling(DOUBLE a)|
DOUBLE|rand(), rand(INT seed)|
DOUBLE|exp(DOUBLE a), exp(DECIMAL a)|
DOUBLE|ln(DOUBLE a), ln(DECIMAL a)|
DOUBLE|log10(DOUBLE a), log10(DECIMAL a)|
DOUBLE|log2(DOUBLE a), log2(DECIMAL a)|
DOUBLE|log(DOUBLE base, DOUBLE a)  log(DECIMAL base, DECIMAL a)|
DOUBLE|pow(DOUBLE a, DOUBLE p)  power(DOUBLE a, DOUBLE p)|
DOUBLE|sqrt(DOUBLE a)  sqrt(DECIMAL a)|
STRING|bin(BIGINT a)|
STRING|hex(BIGINT a)  hex(STRING a)  hex(BINARY a)|
BINARAY|unhex(STRING a)|
STRING|conv(BIGINT num, INT from_base, INT to_base)  conv(STRING num, INT from_base, INT to_base)  DOUBLE|abs(DOUBLE a)|
INT OR DOUBLE|pmod(INT a, INT b)  pmod(DOUBLE a, DOUBLE b)|
DOUBLE|sin(DOUBLE a), sin(DECIMAL a)|
DOUBLE|asin(DOUBLE a), asin(DECIMAL a)|
DOUBLE|cos(DOUBLE a), cos(DECIMAL a)|
DOUBLE|acos(DOUBLE a), acos(DECIMAL a)|
DOUBLE|tan(DOUBLE a), tan(DECIMAL a)|
DOUBLE|atan(DOUBLE a), atan(DECIMAL a)|
DOUBLE|degrees(DOUBLE a), degrees(DECIMAL a)|
DOUBLE|radians(DOUBLE a), radians(DOUBLE a)|
INT OR DOUBLE|positive(INT a)  positive(DOUBLE a)|
INT OR DOUBLE|negative(INT a)  negative(DOUBLE a)|
DOUBLE or INT|sign(DOUBLE a)  sign(DECIMAL a)|
DOUBLE|e()|
DOUBLE|pi()|
BIGINT|factorial(INT a)|
DOUBLT|cbrt(DOUBLE a)|
INT OR BIGINT|shiftleft(TINYINT|SMALLINT|INT a, INT b)  shiftleft(BIGINT a, INT b)|
INT OR BIGINT|shiftright(TINYINT|SMALLINT|INT a, INT b)  shiftright(BIGINT a, INT b)|
INT OR BIGINT|shiftrightunsigned(TINYINT|SMALLINT|INT a, INT b)  shiftrightunsigned(BIGINT a, INT b)|
T|greatest(T v1, T v2, ...)|
T|least(T v1, T v2, ...)|

###### Decimal类型的数学函数和操作符

```
版本：
decimal类型在Hive 0.11.0中引入（[HIVE-2693](https://issues.apache.org/jira/browse/HIVE-2693)）
```
所有常规算术运算符（如+，-，*，/）及相关的算术UDF(Floor, Ceil, Round等)都已更新为处理小数值类型。关于支持的UDF列表，请参考[Mathematical UDFs](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Types#LanguageManualTypes)中的[Hive Data Types](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Types#LanguageManualTypes-MathematicalUDFs)

###### 集合函数

HIVE支持以下内置集合函数

返回类型|名称|描述
-|-|-
int|size(Map<K.V>)
int|size(Array<T>)
array<K>|map_keys(Map<K.V>)
array<V>|map_values(Map<K.V>)
boolean|array_contains(Array<T>, value)
array<t>|sort_array(Array<T>)

###### 类型转换函数

HIVE支持以下类型转换函数

返回类型|名称|描述
-|-|-
binary|binary(string|binary)|
Expected "=" to follow "type"|cast(expr as <type>)|

###### 日期函数

HIVE支持以下日期函数

返回类型|名称|描述
-|-|-
string|from_unixtime(bigint unixtime[, string format])|
bigint|unix_timestamp()|
bigint|unix_timestamp(string date)|
bigint|unix_timestamp(string date, string pattern)|
pre 2.1.0: string  2.1.0 on: date|to_date(string timestamp)|
int|year(string date)|
int|quarter(date/timestamp/string)|
int|month(string date)|
int|day(string date) dayofmonth(date)|
int|hour(string date)|
int|minute(string date)|
int|second(string date)|
int|weekofyear(string date)|
int|datediff(string enddate, string startdate)|
pre 2.1.0: string  2.1.0 on: date|date_add(string startdate, int days)|
pre 2.1.0: string  2.1.0 on: date|date_sub(string startdate, int days)|
timestamp|from_utc_timestamp(timestamp, string timezone)|
timestamp|to_utc_timestamp(timestamp, string timezone)|
date|current_date|
timestamp|current_timestamp|
string|add_months(string start_date, int num_months)|
string|last_day(string date)|
string|next_day(string start_date, string day_of_week)|
string|trunc(string date, string format)
double|months_between(date1, date2)|
string|date_format(date/timestamp/string ts, string fmt)|

###### 条件函数

HIVE支持以下条件函数

返回类型|名称|描述
-|-|-
T|if(boolean testCondition, T valueTrue, T valueFalseOrNull)|
boolean|isnull( a )|
boolean|isnotnull ( a )|
T|nvl(T value, T default_value)|
T|COALESCE(T v1, T v2, ...)|
T|CASE a WHEN b THEN c [WHEN d THEN e]* [ELSE f] END|
T|CASE WHEN a THEN b [WHEN c THEN d]* [ELSE e] END|

###### String函数

HIVE支持以下内置String函数

返回类型|名称|描述
-|-|-
int|ascii(string str)
string|base64(binary bin)
string|concat(string|binary A, string|binary B...)
array<struct<string,double>>|context_ngrams(array<array<string>>, array<string>, int K, int pf)
string|concat_ws(string SEP, string A, string B...)
string|concat_ws(string SEP, array<string>)
string|decode(binary bin, string charset)
binary|encode(string src, string charset)
int|find_in_set(string str, string strList)
string|format_number(number x, int d)
string|get_json_object(string json_string, string path)
boolean|in_file(string str, string filename)
int|instr(string str, string substr)
int|length(string A)
int|locate(string substr, string str[, int pos])
string|lower(string A) lcase(string A)
string|lpad(string str, int len, string pad)
string|ltrim(string A)
array<struct<string,double>>|ngrams(array<array<string>>, int N, int K, int pf)
string|parse_url(string urlString, string partToExtract [, string keyToExtract])
string|printf(String format, Obj... args)
string|regexp_extract(string subject, string pattern, int index)
string|regexp_replace(string INITIAL_STRING, string PATTERN, string REPLACEMENT)
string|repeat(string str, int n)
string|reverse(string A)
string|rpad(string str, int len, string pad)
string|rtrim(string A)
array<array<string>>|sentences(string str, string lang, string locale)
string|space(int n)
array|split(string str, string pat)
map<string,string>|str_to_map(text[, delimiter1, delimiter2])
string|substr(string|binary A, int start)  substring(string|binary A, int start)
string|substr(string|binary A, int start, int len) substring(string|binary A, int start, int len)
string|substring_index(string A, string delim, int count)
string|translate(string|char|varchar input, string|char|varchar from, string|char|varchar to)
string|trim(string A)
binary|unbase64(string str)
string|upper(string A) ucase(string A)
string|initcap(string A)
int|levenshtein(string A, string B)
string|soundex(string A)

###### 其他函数

返回类型|名称|描述
-|-|-
varies|java_method(class, method[, arg1[, arg2..]])
varies|reflect(class, method[, arg1[, arg2..]])
int|hash(a1[, a2...])
string|current_user()
string|current_database()
string|md5(string/binary)
string|sha1(string/binary) sha(string/binary)
bigint|crc32(string/binary)	
string|sha2(string/binary, int)
binary|aes_encrypt(input string/binary, key string/binary)
string|aes_decrypt(input binary, key string/binary)

###### xpath

下列函数在[XPathUDF](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+XPathUDF)中描述:

* xpath, xpath_short, xpath_int, xpath_long, xpath_float, xpath_double, xpath_number, xpath_string

###### get_json_object

JSONPath支持限制：

* $: 
* .:
* []:
* *:

值得注意的是以下语法不支持：

* :
* ..:
* @:
* ():
* ?():
* [,]:
* [start:end.step]:

例如：src_json表仅有一行一列：

```
+----+
                               json
+----+
{"store":
  {"fruit":\[{"weight":8,"type":"apple"},{"weight":9,"type":"pear"}],
   "bicycle":{"price":19.95,"color":"red"}
  },
 "email":"amy@only_for_json_udf_test.net",
 "owner":"amy"
}
+----+
```
可以使用以下查询抽取json对象：

```
hive> SELECT get_json_object(src_json.json, '$.owner') FROM src_json;
amy
 
hive> SELECT get_json_object(src_json.json, '$.store.fruit\[0]') FROM src_json;
{"weight":8,"type":"apple"}
 
hive> SELECT get_json_object(src_json.json, '$.non_exist_key') FROM src_json;
NULL
```

##### 内置聚合函数(UDAF)

Hive支持下内置聚合函数

返回类型|名称|描述
-|-|-
BIGINT|count(*), count(expr), count(DISTINCT expr[, expr...])|
DOUBLE|sum(col), sum(DISTINCT col)|
DOUBLE|avg(col), avg(DISTINCT col)|
DOUBLE|min(col)|
DOUBLE|max(col)
DOUBLE|variance(col), var_pop(col)
DOUBLE|var_samp(col)
DOUBLE|stddev_pop(col)
DOUBLE|stddev_samp(col)
DOUBLE|covar_pop(col1, col2)
DOUBLE|covar_samp(col1, col2)
DOUBLE|corr(col1, col2)
DOUBLE|percentile(BIGINT col, p)
array<double>|percentile(BIGINT col, array(p1 [, p2]...))
DOUBLE|percentile_approx(DOUBLE col, p [, B])
DOUBLE|percentile_approx(DOUBLE col, array(p1 [, p2]...) [, B])
array<struct {'x','y'}>|histogram_numeric(col, b)
array|collect_set(col)
array|collect_list(col)
INTEGER|ntile(INTEGER x)

##### 内置表生成函数(UDTF)

普通的用户自定义函数，如concat()，输入一行输出一行，与此相反，表生成函数将一行转换成多行。

返回类型|名称|描述
-|-|-
N rows|explode(ARRAY)|
N rows|explode(MAP)|
|inline(ARRAY<STRUCT[,STRUCT]>)|
Array Type|explode(array<TYPE> a)|
tuple|json_tuple(jsonStr, k1, k2, ...)|
tuple|parse_url_tuple(url, p1, p2, ...)|
N rows|posexplode(ARRAY)|
N rows|stack(INT n, v_1, v_2, ..., v_k)|

使用语法"SELECT udtf(col) AS colAlias..."有以下限制：

* SELECT子句中不允许其他表达式
	*  SELECT pageid, explode(adid_list) AS myCol... 不支持
* UDTF不能嵌套
	* SELECT explode(explode(adid_list)) AS myCol... 不支持
* GROUP BY / CLUSTER BY / DISTRIBUTE BY / SORT BY不支持
	* SELECT explode(adid_list) AS myCol ... GROUP BY myCol i 不支持

参阅[ LanguageManual LateralView ](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+LateralView)查看替代语法。

如果想创建一个字定义的UDTF，请参阅[ Writing UDTFs ](https://cwiki.apache.org/confluence/display/Hive/DeveloperGuide+UDTF)

###### explode

explode()将一个array(或一个map)作为输入，然后将每个元素作为一行输出。UDTF可用于SELECT表达式列表或作为LATERAL视图的一部分。

SELECT中使用explode()表达式列表的例子，有一个表myTable有一个列myCol和两行数据如下：

Array<int> myCol|
-|
[100,200,300]|
[400,500,600]|

执行这条查询：

```
SELECT explode(myCol) AS myNewCol FROM myTable;
```

将产生如下结果：

(int) myNewCol|
-|
100|
200|
300|
400|
500|
600|

使用MAP类似：

```
SELECT explode(myMap) AS (myMapKey, myMapValue) FROM myMapTable;
```

###### posexplode

```
版本：
Hive 0.13.0可用。查看[HIVE-4943](https://issues.apache.org/jira/browse/HIVE-4943)
```
posexplode()与explode类似，但是它不仅仅返回元素，还会返回元素的位置。

SELECT中使用posexplode()表达式列表的例子，有一个表myTable有一个列myCol和两行数据如下：

Array<int> myCol|
-|
[100,200,300]|
[400,500,600]|

执行如下查询：

```
SELECT posexplode(myCol) AS pos, myNewCol FROM myTable;
```

将产生如下结果：

(int) pos|(int) myNewCol
-|-
1|100
1|400
2|200
2|500
3|300
3|600

###### json_tuple

json_tuple() UDTF于Hive 0.7版本引入，它输入一个names(keys)的set集合和一个JSON字符串，返回一个使用函数的元组。从一个JSON字符串中获取一个以上元素时，这个方法比GET_JSON_OBJECT更有效。在任何一个JSON字符串被解析多次的情况下，查询时只解析一次会更有效率，这就是JSON_TRUPLE的目的。JSON_TUPLE作为一个UDTF，你需要使用LATERAL VIEW语法。

例如：

```
select a.timestamp, get_json_object(a.appevents, '$.eventid'), get_json_object(a.appenvets, '$.eventname') from log a;
```
可以转换为：

```
select a.timestamp, b.*
from log a lateral view json_tuple(a.appevent, 'eventid', 'eventname') b as f1, f2;
```

###### parse_url_tuple

parse_url_tuple() UDTF与parse_url()类似，但是可以抽取指定URL的多个部分，返回一个元组。将key添加在QUERY关键字与：后面，例如：`arse_url_tuple('http://facebook.com/path1/p.php?k1=v1&k2=v2#Ref1', 'QUERY:k1', 'QUERY:k2')`返回一个具有v1和v2的元组。这个方法比多次调用parse_url() 更有效率。所有的输入和输出类型均为string。

```
SELECT b.*
FROM src LATERAL VIEW parse_url_tuple(fullurl, 'HOST', 'PATH', 'QUERY', 'QUERY:id') b as host, path, query, query_id LIMIT 1;
```

##### Grouping and Sorting on

一个典型的OLAP模式：你有一个时间戳列，你想按照daily或者其他更细粒度进行分组。因此你想select concat(year(dt),month(dt)) 然后你想在concat()上做group。但是如果你想要GROUP BY或ORDER BY函数或别名，像这样：

```
select f(col) as fc, count(*) from table_name group by fc;
```

将会产生如下错误：

```
select f(col) as fc, count(*) from table_name group by fc;
```

因为你不能在应用了函数的列别名上使用GROUP BY或ORDER BY，有两个解决办法：
1，使用子查询，看起来有点复杂：

```
select sq.fc,col1,col2,...,colN,count(*) from
  (select f(col) as fc,col1,col2,...,colN from table_name) sq
 group by sq.fc,col1,col2,...,colN;
```
2，你可以确保不使用列别名，这个比较简单：

```
select f(col) as fc, count(*) from table_name group by f(col);
```

##### UDF内部

UDF的上下文计算方法是一次一行，一个简单的使用UDF的例子如下：

```
SELECT length(string_col) FROM table_name;
```
将会在MAP端计算每一个string_col的值的长度。UDF在MAP端进行计算的影响是，不能控制发送给mapper的行的顺序，它与文件片发送给mapper进行反序列化的顺序 相同。任何reduce端的操作（如sort by, order by，普通的join等），将请求UDF的输出就好像它仅仅是表中的另一个列一样。

如果你想控制每一行发送到相同的UDF中（并且排序），你将需要在reduce端进行udf的运算，可以通过 DISTRIBUTE BY, DISTRIBUTE BY + SORT BY, CLUSTER BY来实现，查询例子如下：

```
SELECT reducer_udf(my_col, distribute_col, sort_col) FROM
(SELECT my_col, distribute_col, sort_col FROM table_name DISTRIBUTE BY distribute_col SORT BY distribute_col, sort_col) t
```

然后，值得争议的是，你的需求是控制将行发送到相同的UDF中做聚合，在这种情况下，使用用户定义聚合函数（UDAF）是更好的选择。可以参考更多的关于[UDAF](https://cwiki.apache.org/confluence/display/Hive/GenericUDAFCaseStudy)的写法。也可以使用自定义的[reducer脚本](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Transform)，两者都可以在reduce端做聚合。

##### 创建自定义的UDF

参阅 [Hive Plugins](https://cwiki.apache.org/confluence/display/Hive/HivePlugins)和[Create Function](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-CreateFunction)查看如何创建自定义的UDF.

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

