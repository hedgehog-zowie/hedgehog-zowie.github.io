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
WHERE page_views.date >= '2008-03-01' AND page_views.date &lt;= '2008-03-31'
```

如果表page_views与表dim_users关联，可以像下面一样在on子句中指定分区范围。

```
SELECT page_views.*
FROM page_views JOIN dim_users
  ON (page_views.user_id = dim_users.id AND page_views.date >= '2008-03-01' AND page_views.date &lt;= '2008-03-31')
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
DESCRIBE FUNCTION &lt;function_name>;
DESCRIBE FUNCTION EXTENDED &lt;function_name>;
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
A &lt;=> B|所有基本类型|若操作数非空（non-none），则返回与=相同的结果；若存存空操作数，当两个都为空操作符时，返回TRUE，仅一个为空时，返回FALSE（截至版本[0.9.0](https://issues.apache.org/jira/browse/HIVE-2810)）；
A &lt;> B|所有基本类型|如果A或B为NULL，返回NULL；如果A不等于B，返回TRUE，否则返回FALSE
A != B|所有基本类型|同 &lt;> 
A &lt; B|所有基本类型|如果A或B为NULL，返回NULL；如果A小于B，返回TRUE，否则返回FALSE
A &lt;= B|所有基本类型|如果A或B为NULL，返回NULL；如果A小于等于B，返回TRUE，否则返回FALSE
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
A + B|所有数据类型|返回A与B的和，结果类型是两个操作数的公共父类型。例如：每个int都是float，float是包含int的数据类型，因此一个int和一个float相加其结果是float类型。
A - B|所有数据类型|返回A减B的差，结果类型是两个操作数的公共父类型。
A * B|所有数据类型|返回A与B的乘积，结果类型是两个操作数的公共父类型，注意乘法引起的数据溢出，你需要强制转换为更高精度的数据类型。
A / B|所有数据类型|返回A被B除的结果，大多数情况下其结果类型是double，当A和B均为整型时，若[hive.compat](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-hive.compat)参数设置为『0.13』或『latest』,其结果强制转换为decimal类型，默认该值为『0.12』—— int / int = double，『0.13』—— int / int = decimal。
A % B|所有数据类型|返回A与B取模，结果类型是两个操作数的公共父类型。
A & B|所有数据类型|A和B二进制与，结果类型是两个操作数的公共父类型。
A &#124; B|所有数据类型|A和B二进制或，结果类型是两个操作数的公共父类型。
A ^ B|所有数据类型|A和B二进制异或，结果类型是两个操作数的公共父类型。
~A|所有数据类型|A二进制非（取反），结果类型与A相同。

###### 逻辑运算符

以下操作符支持创建逻辑表达式。所有的返回值均为TRUE，FALSE或NULL，取闷在于操作数的boolean值。NULL标识为"unknown"，因此如果结果取决于"unknown"状态，那么结果本身是"unknown"。

操作符|操作数类型|描述
-|-|-
A AND B|boolean|A与B均为TRUE时为TRUE，否则为FALSE，若A或B为NULL，则返回NULL。
A && B|boolean|同A AND B
A OR B|boolean|A或B为TRUE时为TRUE，FALSE OR NULL 为 NULL, 否则为FALSE。
A &#124;&#124; B|boolean|同 A OR B
NOT A|boolean|A为FALSE时返回TRUE, A为NULL时返回NULL，否则返回FALSE
!A|boolean|同 NOT A
A IN (val1, val2, ...)|boolean|若A与values中的任意元素相同，返回TRUE，HIVE 0.13以上版本支持子查询。
A NOT IN (val1, val2, ...)|boolean|若A与values中的任意元素都不相同，返回TRUE，HIVE 0.13以上版本支持子查询。
[NOT] EXISTS(subquery)||子查询返回至少一行，则为TRUE，Hive 0.13以上版本支持。

###### 复杂类型的运算符

以下函数构造复杂对象的实例。

操作符|操作数|描述
-|-|-
map|(key1, value1, key2, value2, ...)|由给定的key/value对创建一个map
struct|(val1, val2, val3, ...)|创建一个指定元素的struct，字段命名为col1, col2, ...
named_struct|(name1, val1, name2, val2, ...)|创建一个指定名称和元素的struct（Hive 0.8.0以上版本支持）
array|(val1, val2, ...)|创建一个指定元素的数组。
create_union|(tag, val1, val2, ...)|创建一个指定值的union类型。

###### 复杂类型操作

以下函数提供访问复杂对象的机制。

操作符|操作数类型|描述
-|-|-
A[n]|A是一个数组，且n是一个int|返回数组A的第n个元素，第一个元素下标为0，例如果A是['foo','bar']，那么A[0]将返回'foo'，A[1]返回'bar'。
M[key]|M是一个Map&lt;K,V>，且key类型是K|返回map对应key的值，如果M是一个{'f'->'foo','b'->'bar','all'->'foobar'}的map，则M['all']返回'foobar'
S.x|S是一个struct|返回S的x字段，例如struct foobar{int foo, int bar}，foobar.foo返回存储在foo字段的整型值。

##### 内置函数

###### 数学函数

Hive支持以下内置数学函数，当参数为NULL时大部分返回NULL。

返回类型|名称|描述
-|-|-
DOUBLE|round(DOUBLE a)|返回a四舍五入的整数（BIGINT）值。
DOUBLE|round(DOUBLE a, INT d)|返回指定精度的小数，四舍五入。
DOUBLE|bround(DOUBLE a)|返回使用HALF_EVEN的四舍五入（[Hive 1.3.0, 2.0.0以上版本](https://issues.apache.org/jira/browse/HIVE-11103)），也称高斯舍入或银行家舍入：四舍六入五考虑，五后非零就进一，五后为零看奇偶，五前为偶应舍去，五前为奇要进一。
DOUBLE|bround(DOUBLE a, INT d)|返回HALF_EVEN舍入的带精度版，d指定小数位数。
BIGINT|floor(DOUBLE a)|返回小于等于a的最大整数。
BIGINT|ceil(DOUBLE a), ceiling(DOUBLE a)|返回大于等于a的最小整数。
DOUBLE|rand(), rand(INT seed)|返回0到1之间的随机数；指定seed可以确保产生随机数的序列是确定的。
DOUBLE|exp(DOUBLE a), exp(DECIMAL a)|返回e的a次冥，e是自然对数的底数。Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)增加。
DOUBLE|ln(DOUBLE a), ln(DECIMAL a)|返回a的自然对数，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)增加。
DOUBLE|log10(DOUBLE a), log10(DECIMAL a)|返回a以10为底的对数，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)增加。
DOUBLE|log2(DOUBLE a), log2(DECIMAL a)|返回a以2为底的对数，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)增加。
DOUBLE|log(DOUBLE base, DOUBLE a)  log(DECIMAL base, DECIMAL a)|返回a指定底数的对数，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)增加。
DOUBLE|pow(DOUBLE a, DOUBLE p)  power(DOUBLE a, DOUBLE p)|返回a的p次冥
DOUBLE|sqrt(DOUBLE a)  sqrt(DECIMAL a)|返回a的平方根。Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)增加。
STRING|bin(BIGINT a)|返回数值a的二进制格式（查看[更多信息:http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_bin](http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_bin)）
STRING|hex(BIGINT a)  hex(STRING a)  hex(BINARY a)|当参数是一个INT型或binary时，hex返回16进制的字符串；当参数是STRING型时，它将每个字符都转换成16进制，并返回结果字符串。
BINARAY|unhex(STRING a)|16进制的逆，将每对字符看作一个16进制的数，并转换为字节。
STRING|conv(BIGINT num, INT from_base, INT to_base)  conv(STRING num, INT from_base, INT to_base)|进制转换，将数值num从from_base进制转化到to_base进制（[http://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html#function_conv](http://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html#function_conv))
DOUBLE|abs(DOUBLE a)|基于指定的基数，将一个数值转换为另一个，详情请参考[http://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html#function_conv](http://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html#function_conv)。
DOUBLE|abd(DOUBLE a)|返回a的绝对值。
INT OR DOUBLE|pmod(INT a, INT b)  pmod(DOUBLE a, DOUBLE b)|正取余，返回正a对b取模的结果。
DOUBLE|sin(DOUBLE a), sin(DECIMAL a)|正弦，Decima版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|asin(DOUBLE a), asin(DECIMAL a)|反正弦，若 -1 &lt;=a &lt;= 1时，返回a的反正弦值，否则返回NULL。Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|cos(DOUBLE a), cos(DECIMAL a)|余弦，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|acos(DOUBLE a), acos(DECIMAL a)|反余弦，若 -1 &lt;=a &lt;= 1时，返回a的反正弦值，否则返回NULL。Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|tan(DOUBLE a), tan(DECIMAL a)|正切，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|atan(DOUBLE a), atan(DECIMAL a)|反正切，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|degrees(DOUBLE a), degrees(DECIMAL a)|弧度转换成角度
DOUBLE|radians(DOUBLE a), radians(DOUBLE a)|角度转换成弧度
INT OR DOUBLE|positive(INT a)  positive(DOUBLE a)|取正
INT OR DOUBLE|negative(INT a)  negative(DOUBLE a)|取负
DOUBLE or INT|sign(DOUBLE a)  sign(DECIMAL a)|正数返回1.0，负数返回-1.0，否则返回0.0。decimal版本返回INT取代DOUBLE，Decimal版本在[Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-6327)添加
DOUBLE|e()|返回自然常数e的值，约为2.71828，就是公式为lim(1+1/x)^x,x→+∞或lim(1+z)^(1/z),z→0 ，是一个无限不循环小数。
DOUBLE|pi()|返回圆周率pi。
BIGINT|factorial(INT a)|返回a的阶乘（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9858)添加），a的有效取值范围是[0..20]。
DOUBLT|cbrt(DOUBLE a)|返回a的立方根（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9858)添加）。
INT OR BIGINT|shiftleft(TINYINT&#124;SMALLINT&#124;INT a, INT b)  shiftleft(BIGINT a, INT b)|按位左移（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9858)添加），将a左移b个位置，a为tinyint,smallint和int时，返回int型结果；a为bigint时返回bigint型结果。
INT OR BIGINT|shiftright(TINYINT&#124;SMALLINT&#124;INT a, INT b)  shiftright(BIGINT a, INT b)|按位右移（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9858)添加），将a右移b个位置，a为tinyint,smallint和int时，返回int型结果；a为bigint时返回bigint型结果。
INT OR BIGINT|shiftrightunsigned(TINYINT&#124;SMALLINT&#124;INT a, INT b)  shiftrightunsigned(BIGINT a, INT b)|无符号右移（使用0填充最高位）（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9858)添加），将a右移b个位置，a为tinyint,smallint和int时，返回int型结果；a为bigint时返回bigint型结果。
T|greatest(T v1, T v2, ...)|返回指定列表的最大值([Hive 1.1.0](https://issues.apache.org/jira/browse/HIVE-9402)版本添加)，于[Hive 2.0.0](https://issues.apache.org/jira/browse/HIVE-12082)版本修复『当存在一个或多个NULL时，返回NULL的问题，并放宽严格类型要求限制，与">"保持一致』。
T|least(T v1, T v2, ...)|返回指定列表的最小值([Hive 1.1.0](https://issues.apache.org/jira/browse/HIVE-9402)版本添加)，于[Hive 2.0.0](https://issues.apache.org/jira/browse/HIVE-12082)版本修复『当存在一个或多个NULL时，返回NULL的问题，并放宽严格类型要求限制，与"&lt;"保持一致』。

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
int|size(Map&lt;K.V>)|返回map集合的元素数量
int|size(Array&lt;T>)|返回array集合的元素数量
array&lt;K>|map_keys(Map&lt;K.V>)|返回map中的keys的无序数组
array&lt;V>|map_values(Map&lt;K.V>)|返回map中的value的无序数组
boolean|array_contains(Array&lt;T>, value)|如果array中包含值value，则返回true
array&lt;t>|sort_array(Array&lt;T>)|将array按自然顺序排序（[Hive 0.9.0](https://issues.apache.org/jira/browse/HIVE-2279)版本添加）

###### 类型转换函数

HIVE支持以下类型转换函数

返回类型|名称|描述
-|-|-
binary|binary(string&#124;binary)|强制转换为二进制
Expected "=" to follow "type"|cast(expr as &lt;type>)|强制转换表达式expr为类型&lt;type>，例如:cast('1' as BIGINT)将把字符串'1'转换为整型；若转换不成功，将返回null。case(expr as boolean)若expr为非空字符串时，将返回true。

###### 日期函数

HIVE支持以下日期函数

返回类型|名称|描述
-|-|-
string|from_unixtime(bigint unixtime[, string format])|转换自1970-01-01 00:00:00 UTC以来的秒级时间戳为当前时区的时间，格式为 "1970-01-01 00:00:00"。
bigint|unix_timestamp()|获取当前秒级时间戳
bigint|unix_timestamp(string date)|使用当前时区或默认位置，转换指定格式 yyyy-MM-dd HH:mm:ss 的时间字符串为秒级时间戳。若转换失败，则返回0。如：unix_timestamp('2009-03-20 11:30:01') = 1237573801
bigint|unix_timestamp(string date, string pattern)|指定时间字符串的格式(查看[http://docs.oracle.com/javase/tutorial/i18n/format/simpleDateFormat.html](http://docs.oracle.com/javase/tutorial/i18n/format/simpleDateFormat.html)),返回指定日期的秒级时间戳。若转换失败，则返回0。
2.1.0之前: string  2.1.0版本: date|to_date(string timestamp)|返回指定的时间的日期部分，如to_date("1970-01-01 00:00:00") = "1970-01-01"。2.1.0之前返回String，因为该方法创建时，还没有Date类型。
int|year(string date)|返回日期的年部分，如：year("1970-01-01 00:00:00") = 1970, year("1970-01-01") = 1970。
int|quarter(date/timestamp/string)|返回日期的季节（1到4表示）([Hive 1.3.0](https://issues.apache.org/jira/browse/HIVE-3404)添加)，例如：quarter('2015-04-08') = 2。
int|month(string date)|返回月部分。
int|day(string date) dayofmonth(date)|返回日部分。
int|hour(string date)|返回小时部分。
int|minute(string date)|返回分部分。
int|second(string date)|返回秒部分。
int|weekofyear(string date)|返回指定日期是全年的第几个星期。如：weekofyear("1970-11-01 00:00:00") = 44, weekofyear("1970-11-01") = 44.
int|datediff(string enddate, string startdate)|日期间隔，如：datediff('2009-03-01', '2009-02-27') = 2.
pre 2.1.0: string  2.1.0 on: date|date_add(string startdate, int days)|日期增加，如：date_add('2008-12-31', 1) = '2009-01-01'。2.1.0之前返回String，因为该方法创建时，还没有Date类型。
pre 2.1.0: string  2.1.0 on: date|date_sub(string startdate, int days)|日期减法，如：date_sub('2008-12-31', 1) = '2008-12-30'。2.1.0之前返回String，因为该方法创建时，还没有Date类型。
timestamp|from_utc_timestamp(timestamp, string timezone)|假定给定的时间戳是UTC时区，转换为指定的时区([Hive 0.8.0](https://issues.apache.org/jira/browse/HIVE-2272)版本增加)，例如： from_utc_timestamp('1970-01-01 08:00:00','PST') returns 1970-01-01 00:00:00。
timestamp|to_utc_timestamp(timestamp, string timezone)|指定时间和时间戳，将其转换成UTC时区，例如： to_utc_timestamp('1970-01-01 00:00:00','PST') returns 1970-01-01 08:00:00。
date|current_date|返回查询时的当前日期（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-5472)版本添加），同一个查询中所有的current_date返回同一个值。
timestamp|current_timestamp|返回查询时的当前时间戳（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-5472)版本添加），同一个查询中所有的current_timestamp返回同一个值。
string|add_months(string start_date, int num_months)|返回start_date后num_months个月的日期(Hive 1.1.0(https://issues.apache.org/jira/browse/HIVE-9357)版本添加)，时间部分将被忽略，若start_date是一个月的最后一天，而结果月的天数比start_date少，那么结果将返回最后一天。否则，结果的日与start_date相同。
string|last_day(string date)|返回日期所属月的最后一天（[Hive 1.1.0](https://issues.apache.org/jira/browse/HIVE-9358)版本添加），日期字符串格式为 'yyyy-MM-dd HH:mm:ss' 或 'yyyy-MM-dd'，时间部分将被忽略。
string|next_day(string start_date, string day_of_week)|返回start_date后的第一个星期几（day_of_week），day_of_week可以是2个字母、3个字母、或全写，（如Mo, tue, FRIDAY），时间部分将被忽略，例：2015-01-14后的下一个星期二：next_day('2015-01-14', 'TU') = 2015-01-20.
string|trunc(string date, string format)|返回由指定格式截断的日期([Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9480)版本添加)，支持的格式有：MONTH/MON/MM, YEAR/YYYY/YY，例：trunc('2015-03-17', 'MM') = 2015-03-01
double|months_between(date1, date2)|返回date1和date2间隔的月份数量([Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9480)版本添加)，如果date1在date2之后，结果为正数，反之，结果为负数。如果date1和date2的日相同，或者都为月的最后一天，那么结果总是整数，否则，将按31天每月计算。date1和date2可以是date,timestamp或格式为'yyyy-MM-dd'或'yyyy-MM-dd HH:mm:ss'的string。例如：months_between('1997-02-28 10:30:00', '1996-10-30') = 3.94959677
string|date_format(date/timestamp/string ts, string fmt)|格式化日期为fmt格式的字符串，支持[Java SimpleDateFormat格式](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html)，第二个参数fmt是一个常量，例如：date_format('2015-04-08', 'y') = '2015'。date_format能被其他的UDF实现，例如：dayname(date) is date_format(date, 'EEEE')，dayofyear(date) is date_format(date, 'D')

###### 条件函数

HIVE支持以下条件函数

返回类型|名称|描述
-|-|-
T|if(boolean testCondition, T valueTrue, T valueFalseOrNull)|testCondition为true时返回valueTrue，否则返回valueFalseOrNull
boolean|isnull( a )|a为NULL时，返回true,否则返回false
boolean|isnotnull ( a )|a不为NULL时返回true，否则返回false
T|nvl(T value, T default_value)|如果value是null，则返回default value，否则返回value([Hive 0.11](https://issues.apache.org/jira/browse/HIVE-2288)版本添加)。
T|COALESCE(T v1, T v2, ...)|返回第一个非NULL值，或者当或者值均为NULL时返回NULL。
T|CASE a WHEN b THEN c [WHEN d THEN e]* [ELSE f] END|When a = b, returns c; when a = d, returns e; else returns f.
T|CASE WHEN a THEN b [WHEN c THEN d]* [ELSE e] END|When a = true, returns b; when c = true, returns d; else returns e.

###### String函数

HIVE支持以下内置String函数

返回类型|名称|描述
-|-|-
int|ascii(string str)|返回str第一个字符的数值。
string|base64(binary bin)|转换二进制为base 64的字符串([Hive 0.12.0](https://issues.apache.org/jira/browse/HIVE-2482)版本添加)
string|chr(bigint&#124;double A)|返回A的ASCII码([Hive 1.3.0 和 Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13063)版本添加)，如果A比256大，则结果与chr(A % 256)等价，例如：select chr(88); 返回"X"。
string|concat(string&#124;binary A, string&#124;binary B...)|连接字符串，例如： concat('foo', 'bar') 结果是 'foobar'，注意这个函数可以接任意多个输入。
array&lt;struct&lt;string,double>>|context_ngrams(array&lt;array&lt;string>>, array&lt;string>, int K, int pf)|返回array&lt;array&lt;string>>中出现在array&lt;string>后最频繁的top-k个词及其出现次数，详情请参考[StatisticsAndDataMining](https://cwiki.apache.org/confluence/display/Hive/StatisticsAndDataMining)。
string|concat_ws(string SEP, string A, string B...)|同concat类似，但是可以由用户指定分隔符SEP。
string|concat_ws(string SEP, array&lt;string>)|同concat_ws，但是参数是一个string型的字符串。
string|decode(binary bin, string charset)|根据character（'US-ASCII', 'ISO-8859-1', 'UTF-8', 'UTF-16BE', 'UTF-16LE', 'UTF-16'其中之一）指定的编码方式进行解码，如果任一参数是null，结果为null.([Hive 0.12.0](https://issues.apache.org/jira/browse/HIVE-2482)版本添加)
binary|encode(string src, string charset)|根据character（'US-ASCII', 'ISO-8859-1', 'UTF-8', 'UTF-16BE', 'UTF-16LE', 'UTF-16'其中之一）指定的编码方式进行编码，如果任一参数是null，结果为null.([Hive 0.12.0](https://issues.apache.org/jira/browse/HIVE-2482)版本添加)
int|find_in_set(string str, string strList)|strList是一个逗号分隔的字符串，返回第一个str出现的位置。若任一参数为null，则返回null。若第一个参数包含任何逗号，则返回0.例如：find_in_set('ab', 'abc,b,ab,c,def') 返回 3.
string|format_number(number x, int d)|格式化数值x，如'#,###,###.##', 根据d指定的精度四舍五入，并返回一个字符串。如果d是0 ,结果没有小数点和小数部分。([Hive 0.10.0](https://issues.apache.org/jira/browse/HIVE-2694)版本添加，[Hive 0.14.0](https://issues.apache.org/jira/browse/HIVE-7257)版本修复float类型bug，[Hive 0.14.0](https://issues.apache.org/jira/browse/HIVE-7279)版本支持decimal类型。)
string|get_json_object(string json_string, string path)|从json字符串中提取由json path指定的json对象，如果json字符串无效，则返回null。注意：json path仅可拥有字符[0-9a-z_]，没有大写和特殊字符。而且，keys不能以数字开头。这是由Hive列名约束的。
boolean|in_file(string str, string filename)|若str在文件filename中以一整行的形式出现，则返回true。
int|instr(string str, string substr)|返回str中第一次出现substr的位置，任一参数为null，则返回null。如果substr没有在str中出现，则返回0。注意，这个函数不是基于0的，str中的第一个字符是1.
int|length(string A)|返回string的长度。
int|locate(string substr, string str[, int pos])|返回position位置之后第一次出现substr的位置。
string|lower(string A) lcase(string A)|转换所有字符 为小写。如：lower('fOoBaR') 结果为 'foobar'。
string|lpad(string str, int len, string pad)|左侧填充长度len的pad字符串
string|ltrim(string A)|去除左侧空白字符，例如：ltrim(' foobar ') 结果为 'foobar '。
array&lt;struct&lt;string,double>>|ngrams(array&lt;array&lt;string>>, int N, int K, int pf)|返回array&lt;array&lt;string>>中出现连续的N个字符最频繁的top-k个词及其出现次数，详情请参考[StatisticsAndDataMining](https://cwiki.apache.org/confluence/display/Hive/StatisticsAndDataMining)。
string|parse_url(string urlString, string partToExtract [, string keyToExtract])|返回url的指定部分。有效的partToExtract包括：HOST, PATH, QUERY, REF, PROTOCOL, AUTHORITY, FILE 和 USERINFO。例如：parse_url('http://facebook.com/path1/p.php?k1=v1&k2=v2#Ref1', 'HOST') 返回 'facebook.com'。并且，关键字QUERY，可以提取第三个参数指定的参数值，例如：parse_url('http://facebook.com/path1/p.php?k1=v1&k2=v2#Ref1', 'QUERY', 'k1') 返回 'v1'。
string|printf(String format, Obj... args)|返回输入格式对应的打印格式([Hive 0.9.0](https://issues.apache.org/jira/browse/HIVE-2695)版本添加)。
string|regexp_extract(string subject, string pattern, int index)|返回使用pattern提取的字符串。例如：regexp_extract('foothebar', 'foo(.*?)(bar)', 1) 返回 'the'， regexp_extract('foothebar', 'foo(.*?)(bar)', 2) 返回 'bar'，regexp_extract('foothebar', 'foo(.*?)(bar)', 0) 返回 'foothebar', 注意一些情况下需要使用转义字符：使用'\s'作为第二个参数将匹配字母s；'\\s'匹配空白字符等。'index'参数是Java regex Maatcher group()方法的index。查看docs/api/java/util/regex/Matcher.html获取更多信息。
string|regexp_replace(string INITIAL_STRING, string PATTERN, string REPLACEMENT)|用REPLACEMENT替换INITIAL_STRING中所有匹配PATTERN的字符串，例如：regexp_replace("foobar", "oo&#124;ar", "") 返回 'fb'。注意一些情况下需要使用转义字符：使用'\s'作为第二个参数将匹配字母s；'\\s'匹配空白字符等。
string|repeat(string str, int n)|重复str字符串n次。
string|replace(string A, string OLD, string NEW)|使用NEW替换A中所有不重叠的OLD字符串（[Hive 1.3.0 和 Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13063)版本添加）,例如：select replace("ababab", "abab", "Z"); 返回 "Zab".
string|reverse(string A)|字符串反转。
string|rpad(string str, int len, string pad)|右侧填充长度len的pad字符串
string|rtrim(string A)|去除左侧空白字符，例如：ltrim(' foobar ') 结果为 ' foobar'。
array&lt;array&lt;string>>|sentences(string str, string lang, string locale)|分隔自然语言文本为单词和句子，lang和locale参数可选，例如：sentences('Hello there! How are you?') 返回 ( ("Hello", "there"), ("How", "are", "you") )
string|space(int n)|返回n个空格的字符串
array|split(string str, string pat)|用pat分隔字符串，pat是一个普通的表达式。
map&lt;string,string>|str_to_map(text[, delimiter1, delimiter2])|用两个分隔符将字符串分割为key-value键值对，第一个分隔符delimiter1将字符串分隔为键值对字符串，第二个分隔符delimiter2将每一个键值对字符串分隔为键值对。
string|substr(string&#124;binary A, int start)  substring(string&#124;binary A, int start)|从start位置开始，获取A的子串
string|substr(string&#124;binary A, int start, int len)  substring(string&#124;binary A, int start, int len)|从start位置开始获取长度为len的A的子串
string|substring_index(string A, string delim, int count)|返回A中由delim分隔的count位置的字符串。如果count为正数，返回从左到最后一个分隔符（从左往右数）的字符串，如果count为负数，返回从右到最后一个分隔符（从右往左数）的字符串。例如：substring_index('www.apache.org', '.', 2) = 'www.apache'.
string|translate(string&#124;char&#124;varchar input, string&#124;char&#124;varchar from, string&#124;char&#124;varchar to)|将input中出现在from中的字符替换为to中的字符串，同PostgreSQL中的translate函数，如果任何参数为null，结果为null。（[Hive 0.10.0](https://issues.apache.org/jira/browse/HIVE-2418)以上版本可用，支持string类型）,[Hive 0.14.0](https://issues.apache.org/jira/browse/HIVE-6622)以上版本支持Char/varchar类型
string|trim(string A)|去除两侧空白字符，例如：trim(' foobar ') 结果 是 'foobar'
binary|unbase64(string str)|转换base 64字符串为二进制（[Hive 0.12.0](https://issues.apache.org/jira/browse/HIVE-2482)版本添加）
string|upper(string A) ucase(string A)|所有字符大写，例如：upper('fOoBaR') 结果为 'FOOBAR'。
string|initcap(string A)|首字母大写，其他字符小写。单词由空白字符分隔。([Hive 1.1.0](https://issues.apache.org/jira/browse/HIVE-3405)版本添加)
int|levenshtein(string A, string B)|返回两个字符串之间的编辑距离（是指两个字串之间，由一个转成另一个所需的最少编辑操作次数），如： levenshtein('kitten', 'sitting') 结果为 3。([Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9556)版本添加)
string|soundex(string A)|SOUNDEX返回由四个字符组成的代码 (SOUNDEX) 以评估两个字符串的相似性, 代码的第一个字符是 A 的第一个字符，代码的第二个字符到第四个字符是数字。例如：soundex('Miller') 结果为 M460。（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9738)版本添加）

###### 数据掩饰函数 

Hive支持以下数据掩饰函数。

返回类型|名称|描述
-|-|-
string|mask(string str[, string upper[, string lower[, string number]]])|返回一个掩饰后的字符串（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13568)添加），默认情况下，大写字符转换成"X"，小写字符转换成"x"，数字转换成"n"。例如：mask("abcd-EFGH-8765-4321") 结果为 xxxx-XXXX-nnnn-nnnn。你可以使用提供的附加参数复写替换的参数值，第二个参数掩饰大写字符 ，第三个参数掩饰小写字符，第四个参数掩饰数字，例如：mask("abcd-EFGH-8765-4321", "U", "l", "#") 结果是 llll-UUUU-####-####.
string|mask_first_n(string str[, int n])|掩饰数字串str的头n个字符，例如：mask_first_n("1234-5678-8765-4321", 4) results in nnnn-5678-8765-4321。（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13568)添加）
string|mask_last_n(string str[, int n])|掩饰数字串str的最后n个字符，例如：mask_last_n("1234-5678-8765-4321", 4) results in 1234-5678-8765-nnnn。（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13568)添加）
string|mask_show_first_n(string str[, int n])|仅显示头n个字符，其他字符都做掩饰，例如：mask_show_first_n("1234-5678-8765-4321", 4) results in 1234-nnnn-nnnn-nnnn。（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13568)添加）
string|mask_show_last_n(string str[, int n])|显示最后n个字符，其他字符都做掩饰，例如：mask_show_last_n("1234-5678-8765-4321", 4) results in nnnn-nnnn-nnnn-4321。（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13568)添加）
string|mask_hash(string&#124;char&#124;varchar str)|返回str的hash值，该hash值是唯一的，并且可以用于表与表之间的关联。若str字符串为空字符串，该函数返回null。（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-13568)添加）

###### 混合函数

返回类型|名称|描述
-|-|-
varies|java_method(class, method[, arg1[, arg2..]])|reflect的同义词（[Hive 0.9.0(https://issues.apache.org/jira/browse/HIVE-1877)]版本添加）
varies|reflect(class, method[, arg1[, arg2..]])|使用反射调用匹配参数的Java方法，查看[Reflect (Generic) UDF](https://cwiki.apache.org/confluence/display/Hive/ReflectUDF)
int|hash(a1[, a2...])|返回参数的hash值(Hive 0.4.)
string|current_user()|返回当前用户（[Hive 1.2.0](https://issues.apache.org/jira/browse/HIVE-9143)）
string|current_database()|返回当前数据库名([Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-4144))
string|md5(string/binary)|计算128位的MD5校验码。（[Hive 1.3.0](https://issues.apache.org/jira/browse/HIVE-10485)版本添加），该值以一个32位的16进制码返回。若参数为NULL则返回NULL，例如：md5('ABC') = '902fbdd2b1df0c4f70b4a5d23525e932'。
string|sha1(string/binary) sha(string/binary)|计算SHA-1消息摘要，并返回16进制字符串（[Hive 1.3.0](https://issues.apache.org/jira/browse/HIVE-10485)版本添加），例如：sha1('ABC') = '3c01bdbb26f358bab27f267924aa2c9a03fcfdb8'.
bigint|crc32(string/binary)	|计算循环冗余校验码，返回bigint型的数值。例如：crc32('ABC') = 2743272264。
string|sha2(string/binary, int)|计算SHA-2家族的hash函数（SHA-224, SHA-256, SHA-384, and SHA-512）([Hive 1.3.0](https://issues.apache.org/jira/browse/HIVE-10644)添加)，第一个参数是需要hash的字符串或二进制，第二个参数是结果长度，必须是224, 256, 384, 512, or 0 (等价于256)，SHA-224从Java 8开始支持。如果任一参数为NULL，或者hash长度无效，则返回NULL。例如：sha2('ABC', 256) = 'b5d4045c3f466fa91fe2cc6abe79232a1a57cdf104f7a26e716e0a1e2789df78'。
binary|aes_encrypt(input string/binary, key string/binary)|AES加密（[Hive 1.3.0(https://issues.apache.org/jira/browse/HIVE-11593)]），key的长度是128，192或256。192和256在安装了Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files的情况下才适用。如果任一参数为NULL，或者hash长度无效，则返回NULL。例如：base64(aes_encrypt('ABC', '1234567890123456')) = 'y6Ss+zCYObpCbgfWfyNWTw=='。
string|aes_decrypt(input binary, key string/binary)|AES解密，key的长度是128，192或256。192和256在安装了Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files的情况下才适用。如果任一参数为NULL，或者hash长度无效，则返回NULL。例如：aes_decrypt(unbase64('y6Ss+zCYObpCbgfWfyNWTw=='), '1234567890123456') = 'ABC'。
string|version()|返回Hive版本。（[Hive 2.1.0](https://issues.apache.org/jira/browse/HIVE-12983)版本支持）。结果包含2个字段 ，第一个字段是一个数字 ，第二个字段是一段hash码。例如："select version();" 可能返回 "2.1.0.2.5.0.0-1245 r027527b9c5ce1a3d7d0b6d2e6de2378fb0c39232"，事实上，返回结果依赖于你的构建。

###### xpath

下列函数在[XPathUDF](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+XPathUDF)中描述:

* xpath, xpath_short, xpath_int, xpath_long, xpath_float, xpath_double, xpath_number, xpath_string

###### get_json_object

JSONPath支持限制：

* $: 根对象
* .: 子对象
* []:数组下标
* *:通配符

值得注意的是以下语法不支持：

* :0长度的字符串作为key
* ..:递归 
* @:当前对象
* ():脚本表达式
* ?():过滤表达式
* [,]:Union操作符
* [start:end.step]:数组分隔符

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
BIGINT|count(*), count(expr), count(DISTINCT expr[, expr...])|count(*)返回检索的总行数，包括值为NULL的行；count(expr)返回表达式expr不为NULL的总行数；count(DISTINCT expr[, expr]) 返回表达式expr唯一也不为NULL的总行数，使用[hive.optimize.distinct.rewrite](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-hive.optimize.distinct.rewrite)执行该函数将得到更好的执行效率。
DOUBLE|sum(col), sum(DISTINCT col)|返回组内的某列的和，或者是组内某列唯一值的和。
DOUBLE|avg(col), avg(DISTINCT col)|返回每个分组的某列平均值，或者是组内某列唯一值的平均值。
DOUBLE|min(col)|每个组内最小值。
DOUBLE|max(col)|每个组内最大值。
DOUBLE|variance(col), var_pop(col)|组内一个数值列的方差。
DOUBLE|var_samp(col)|组内一个数值列的无偏样本方差。
DOUBLE|stddev_pop(col)|组内一个数值列的标准差。
DOUBLE|stddev_samp(col)|组内一个数值列的无偏样子标准差。
DOUBLE|covar_pop(col1, col2)|组内两个数值列的总体协方差。
DOUBLE|covar_samp(col1, col2)|组内两个数值列的样本协方差。
DOUBLE|corr(col1, col2)|组内两个数值列的皮尔逊相关系数。
DOUBLE|percentile(BIGINT col, p)|组内一个列精确到p位的百分数，p必须在0和1之间。
array&lt;double>|percentile(BIGINT col, array(p1 [, p2]...))|组内一个列精确的第p1,p2,...位百分数，p必须在0和1之间。
DOUBLE|percentile_approx(DOUBLE col, p [, B])|组内一个数值列的第p位百分数（包括浮点数），参数B控制近似的精确度，B值越大，近似度越高，默认值为10000。当列中非重复值的数量小于B时，返回精确的百分数
array&lt;double>|percentile_approx(DOUBLE col, array(p1 [, p2]...) [, B])|同上，但接受并返回百分数数组
array&lt;struct {'x','y'}>|histogram_numeric(col, b)|使用b个非均匀间隔的箱子计算组内数字列的柱状图（直方图），输出的数组大小为b，double类型的(x,y)表示直方图的中心和高度
array|collect_set(col)|返回消除了重复元素的数组
array|collect_list(col)|返回允许重复元素的数组
INTEGER|ntile(INTEGER x)|该函数将已经排序的分区分到x个桶中，并为每行分配一个桶号。这可以容易的计算三分位，四分位，十分位，百分位和其它通用的概要统计

##### 内置Table-Generating函数(UDTF)

普通的用户自定义函数，如concat()，输入一行输出一行，与此相反，表生成函数将一行转换成多行。

返回类型|名称|描述
-|-|-
N rows|explode(ARRAY)|将数组数据中的每个元素做为一行返回
N rows|explode(MAP)|将输入map中的每个键值对转换为两列，一列为key，另一列为value，然后返回新行（[Hive 0.8.0](https://issues.apache.org/jira/browse/HIVE-1735)版本支持）
|inline(ARRAY&lt;STRUCT[,STRUCT]>)|分解struct数组到表中（[Hive 0.10(https://issues.apache.org/jira/browse/HIVE-3238)]）
Array Type|explode(array&lt;TYPE> a)|对于数组a中的每个元素，产生包含该元素的行
tuple|json_tuple(jsonStr, k1, k2, ...)|参数为一组键k1，k2……和JSON字符串，返回值的元组。该方法比 get_json_object 高效，因为可以在一次调用中输入多个键
tuple|parse_url_tuple(url, p1, p2, ...)|该方法同parse_url() 相似，但可以一次性提取URL的多个部分，有效的参数名称为： HOST, PATH, QUERY, REF, PROTOCOL, AUTHORITY, FILE, USERINFO, QUERY:&lt;KEY>
N rows|posexplode(ARRAY)|行为与参数为数组的explode方法相似，但包含项在原始数组中的位置，返回(pos,value)的二元组。([Hive 0.13.0](https://issues.apache.org/jira/browse/HIVE-4943))
N rows|stack(INT n, v_1, v_2, ..., v_k)|将v_1, ..., v_k 分为n行，每行包含k/n列，n必须为常数

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

Array&lt;int> myCol|
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

Array&lt;int> myCol|
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

##### UDFs

###### xpath, xpath_short, xpath_int, xpath_long, xpath_float, xpath_double, xpath_number, xpath_string

* 使用XPath表达式解析XML数据。
* 起始版本：0.6.0

###### 概述

xpath UDFs家族封装了JDK提供的javax.xml.xpath库，该库基于XPath 1.0，获取更多Java XPath library信息[http://java.sun.com/javase/6/docs/api/javax/xml/xpath/package-summary.html](http://java.sun.com/javase/6/docs/api/javax/xml/xpath/package-summary.html)。

所有的函数格式都是：xpath_*(xml_string, xpath_expression_string)，XPath表达式会被编译和缓存。如果下一个输入匹配以前的表达式，则重用。否则，重新编译。因此，xml字符串解析每一行输入，而xpath表达式预编译并重用大部分的使用场景。

例如：

```
> select xpath ('&lt;a>&lt;b id="1">&lt;c/>&lt;/b>&lt;b id="2">&lt;c/>&lt;/b>&lt;/a>','/descendant::c/ancestor::b/@id') from t1 limit 1 ;
[1","2]
```

每一个函数返回一个指定的Hive类型：

* xpath返回一个hive字符串数组。
* xpath_string返回string。
* xpath_boolean返回boolean。
* xpath_short返回short integer
* xpath_int返回integer。
* xpath_long返回long integer。
* xpath_float返回floating point number.
* xpath_double,xpath_number返回double-precision floating point number (xpath_number是xpath_double的别名).

错误的xml格式（如：&lt;a>&lt;b>1&lt;/b>&lt;/aa>）将会引起运行时异常。

下面说明不同的xpath UDF变种。

###### xpath

xpath()函数总是返回string数组，如果表达式结果是非文本的值（如:另一个xml节点），该函数将返回空数组[]，该函数有两个主要的用途：获得一个文本值列表或一个属性值列表。

示列：
匹配到空：
```
> select xpath('&lt;a>&lt;b>b1&lt;/b>&lt;b>b2&lt;/b>&lt;/a>','a/*') from src limit 1 ;
[]
```

获得一个节点列表：
```
> select xpath('&lt;a>&lt;b>b1&lt;/b>&lt;b>b2&lt;/b>&lt;/a>','a/*/text()') from src limit 1 ;
[b1","b2]
```

获得id属性值的列表：
```
> select xpath('&lt;a>&lt;b id="foo">b1&lt;/b>&lt;b id="bar">b2&lt;/b>&lt;/a>','//@id') from src limit 1 ;
[foo","bar]
```

获取class属性为'bb'的节点列表：
```
> SELECT xpath ('&lt;a>&lt;b class="bb">b1&lt;/b>&lt;b>b2&lt;/b>&lt;b>b3&lt;/b>&lt;c class="bb">c1&lt;/c>&lt;c>c2&lt;/c>&lt;/a>', 'a/*[@class="bb"]/text()') FROM src LIMIT 1 ;
[b1","c1]
```

###### xpath_string

xpath_string()函数返回每一个匹配的节点：

获得节点'a/b':
```
> SELECT xpath_string ('&lt;a>&lt;b>bb&lt;/b>&lt;c>cc&lt;/c>&lt;/a>', 'a/b') FROM src LIMIT 1 ;
bb
```

获得节点a，由于节点a存在子节点，结果合成子节点的文本：
```
> SELECT xpath_string ('&lt;a>&lt;b>bb&lt;/b>&lt;c>cc&lt;/c>&lt;/a>', 'a') FROM src LIMIT 1 ;
bbcc
```

不匹配时返回空字符串：
```
> SELECT xpath_string ('&lt;a>&lt;b>bb&lt;/b>&lt;c>cc&lt;/c>&lt;/a>', 'a/d') FROM src LIMIT 1 ;
```

返回匹配'//b'的第一个节点：
```
> SELECT xpath_string ('&lt;a>&lt;b>b1&lt;/b>&lt;b>b2&lt;/b>&lt;/a>', '//b') FROM src LIMIT 1 ;
b1
```

获取第二个节点：
```
> SELECT xpath_string ('&lt;a>&lt;b>b1&lt;/b>&lt;b>b2&lt;/b>&lt;/a>', 'a/b[2]') FROM src LIMIT 1 ;
b2
```

获取第一个拥有属性id且值为b_2的节点文本：
```
> SELECT xpath_string ('&lt;a>&lt;b>b1&lt;/b>&lt;b id="b_2">b2&lt;/b>&lt;/a>', 'a/b[@id="b_2"]') FROM src LIMIT 1 ;
b2
```

###### xpath_boolean

如果XPath表达式为true或发现匹配的节点，则返回true。

匹配节点：
```
> SELECT xpath_boolean ('&lt;a>&lt;b>b&lt;/b>&lt;/a>', 'a/b') FROM src LIMIT 1 ;
true
```

不匹配节点：
```
> SELECT xpath_boolean ('&lt;a>&lt;b>b&lt;/b>&lt;/a>', 'a/c') FROM src LIMIT 1 ;
false
```

匹配表达式：
```
> SELECT xpath_boolean ('&lt;a>&lt;b>b&lt;/b>&lt;/a>', 'a/b = "b"') FROM src LIMIT 1 ;
true
```

不匹配表达式：
```
> SELECT xpath_boolean ('&lt;a>&lt;b>10&lt;/b>&lt;/a>', 'a/b &lt; 10') FROM src LIMIT 1 ;
false
```

###### xpath_short, xpath_int, xpath_long

这些函数返回一个整型数值，若不匹配或匹配但值为非数值时，则返回0。支持数学运算。在返回值数据溢出时，返回该类型的最大值。

不匹配：
```
> SELECT xpath_int ('&lt;a>b&lt;/a>', 'a = 10') FROM src LIMIT 1 ;
0
```

非数值：
```
> SELECT xpath_int ('&lt;a>this is not a number&lt;/a>', 'a') FROM src LIMIT 1 ;
0
> SELECT xpath_int ('&lt;a>this 2 is not a number&lt;/a>', 'a') FROM src LIMIT 1 ;
0
```

增加值：
```
> SELECT xpath_int ('&lt;a>&lt;b class="odd">1&lt;/b>&lt;b class="even">2&lt;/b>&lt;b class="odd">4&lt;/b>&lt;c>8&lt;/c>&lt;/a>', 'sum(a/*)') FROM src LIMIT 1 ;
15
> SELECT xpath_int ('&lt;a>&lt;b class="odd">1&lt;/b>&lt;b class="even">2&lt;/b>&lt;b class="odd">4&lt;/b>&lt;c>8&lt;/c>&lt;/a>', 'sum(a/b)') FROM src LIMIT 1 ;
7
> SELECT xpath_int ('&lt;a>&lt;b class="odd">1&lt;/b>&lt;b class="even">2&lt;/b>&lt;b class="odd">4&lt;/b>&lt;c>8&lt;/c>&lt;/a>', 'sum(a/b[@class="odd"])') FROM src LIMIT 1 ;
5
```

数据溢出：
```
> SELECT xpath_int ('&lt;a>&lt;b>2000000000&lt;/b>&lt;c>40000000000&lt;/c>&lt;/a>', 'a/b * a/c') FROM src LIMIT 1 ;
2147483647
```

###### xpath_float, xpath_double, xpath_number

同 xpath_short, xpath_int 和 xpath_long一样，但是返回浮点数。不匹配时结果为0，然后匹配到非0值是结果为NaN。注意xpath_number()是xpath_double()的别名。

不匹配：
```
> SELECT xpath_double ('&lt;a>b&lt;/a>', 'a = 10') FROM src LIMIT 1 ;
0.0
```

非数值：
```
> SELECT xpath_double ('&lt;a>this is not a number&lt;/a>', 'a') FROM src LIMIT 1 ;
NaN
```

一个非常大的数：
```
SELECT xpath_double ('&lt;a>&lt;b>2000000000&lt;/b>&lt;c>40000000000&lt;/c>&lt;/a>', 'a/b * a/c') FROM src LIMIT 1 ;
8.0E19
```

##### UDAFs

##### UDTFs

#### Joins

##### Join语法

Hive支持以下join语法：

```
join_table:
    table_reference JOIN table_factor [join_condition]
  | table_reference {LEFT|RIGHT|FULL} [OUTER] JOIN table_reference join_condition
  | table_reference LEFT SEMI JOIN table_reference join_condition
  | table_reference CROSS JOIN table_reference [join_condition] (as of Hive 0.10)
  
table_reference:
    table_factor
  | join_table
  
table_factor:
    tbl_name [alias]
  | table_subquery alias
  | ( table_references )
  
join_condition:
    ON equality_expression ( AND equality_expression )*
    
equality_expression:
    expression = expression
```

Hive中仅支持equality joins, outer joins, 和left semi joins，Hive不支持不相等条件关联，因为在map/reduce中要表示这种条件非常困难。另外，Hive支持两个表以上的关联。

查看[Select Syntax](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select#LanguageManualSelect-SelectSyntax)更多的join语法。

```
0.13.0以上版本：隐式关联
隐式关联在Hive 0.13.0(查看 [HIVE-5558](https://issues.apache.org/jira/browse/HIVE-5558))以上版本支持，允许关联FROM子句的逗号分隔的表，省略JOIN关键字，例如：
SELECT * 
FROM table1 t1, table2 t2, table3 t3 
WHERE t1.id = t2.id AND t2.id = t3.id AND t1.zipcode = '02535';
```

```
0.13.0以上版本：未指定的列引用
join条件支持未指定列引用，在Hive 0.13.0(查看[HIVE-6393](https://issues.apache.org/jira/browse/HIVE-6393))。Hive尝试解析这些输入，如果一个未指定的列引用解析到匹配多张表，Hive会标记它为一个模糊的引用。例如：
CREATE TABLE a (k1 string, v1 string);
CREATE TABLE b (k2 string, v2 string);
SELECT k1, v1, k2, v2
FROM a JOIN b ON k1 = k2; 
```

示例：
在写join查询时，一些突出的点尤其值得注意：

* 仅支持equality join如下：
```
SELECT a.* FROM a JOIN b ON (a.id = b.id)
```
```
SELECT a.* FROM a JOIN b ON (a.id = b.id AND a.department = b.department)
```
都是有效的join，然而：
```
SELECT a.* FROM a JOIN b ON (a.id &lt;> b.id)
```
是不允许的。

* 同一个查询中2张以上表关联：
```
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
```
是有效的join。

* 如果每张表有相同的列用于join子句，Hive将转换多个表间的join操作为一个单独的map/reduce job，例如：
```
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
```
被转化为一个单独的map/reduce job，因为仅有表b的key1列需要join，而另一个处理：
```
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
```
被转换为两个map/reduce job，因为表b的key1列用于第一个join条件，而key2列用于第二个。第一个map/reduce job关联a和b，其结果再在第二个map/reduce job中关联c表。

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

