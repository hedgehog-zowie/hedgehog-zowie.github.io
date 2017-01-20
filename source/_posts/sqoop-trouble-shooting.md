---
title: sqoop-trouble-shooting
date: 2016-10-14 17:37:51
toc: true
tags:
- sqoop
categories: 
- sqoop
---

记录使用sqoop过程中遇到的问题、原因及解决办法。

常用命令示例：
导入HDFS：
`bin/sqoop import --connect jdbc:mysql://10.10.10.84:3306/test --table czw_test_distinct --username ---------- --password ---------- --target-dir /tmp/testSqoop --driver com.mysql.jdbc.Driver --split-by a`
导入Hive:
`bin/sqoop import --connect jdbc:mysql://10.10.10.84:3306/test --table czw_test_distinct --username ---------- --password ---------- --hive-import --driver com.mysql.jdbc.Driver --split-by a`

导出(hive -> mysql):
`bin/sqoop export --connect jdbc:mysql://10.10.10.84:3306/test --table czw_test_distinct --username ---------- --password ---------- --driver com.mysql.jdbc.Driver --export-dir /user/hive/warehouse/czw_test_distinct --input-fields-terminated-by '\001'`
或(hdfs -> mysql)：
`bin/sqoop export --connect jdbc:mysql://10.10.10.84:3306/test --table czw_test_distinct --username ---------- --password ---------- --driver com.mysql.jdbc.Driver --export-dir /tmp/testSqoop --input-fields-terminated-by ','`

导入rcfile表
`bin/sqoop import --connect jdbc:mysql://10.10.10.84:3306/test --table test_sqoop_rcfile_to_mysql --username ---------- --password ---------- --driver com.mysql.jdbc.Driver --hcatalog-table test_sqoop_rcfile_to_mysql --split-by app_id`
导出rcfile表
`bin/sqoop export --connect jdbc:mysql://10.10.10.84:3306/test --table test_sqoop_rcfile_to_mysql --username ---------- --password ---------- --driver com.mysql.jdbc.Driver --hcatalog-table test_sqoop_rcfile_to_mysql`


# No columns to generate for ClassWriter

执行如下命令导入mysql数据到hadoop：
`bin/sqoop import --connect jdbc:mysql://10.10.10.84:3306/test --table czw_test_distinct --username ---------- --password ---------- --target-dir /tmp/testSqoop`

出现如下错误

```
16/11/28 14:30:27 ERROR manager.SqlManager: Error reading from database: java.sql.SQLException: Streaming result set com.mysql.jdbc.RowDataDynamic@24aa8801 is still active. No statements may be issued when any streaming result sets are open and in use on a given connection. Ensure that you have called .close() on any active streaming result sets before attempting more queries.
java.sql.SQLException: Streaming result set com.mysql.jdbc.RowDataDynamic@24aa8801 is still active. No statements may be issued when any streaming result sets are open and in use on a given connection. Ensure that you have called .close() on any active streaming result sets before attempting more queries.
        at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:930)
        at com.mysql.jdbc.MysqlIO.checkForOutstandingStreamingData(MysqlIO.java:2646)
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:1861)
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2101)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2548)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2477)
        at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1422)
        at com.mysql.jdbc.ConnectionImpl.getMaxBytesPerChar(ConnectionImpl.java:2945)
        at com.mysql.jdbc.Field.getMaxBytesPerCharacter(Field.java:582)
        at com.mysql.jdbc.ResultSetMetaData.getPrecision(ResultSetMetaData.java:441)
        at org.apache.sqoop.manager.SqlManager.getColumnInfoForRawQuery(SqlManager.java:286)
        at org.apache.sqoop.manager.SqlManager.getColumnTypesForRawQuery(SqlManager.java:241)
        at org.apache.sqoop.manager.SqlManager.getColumnTypes(SqlManager.java:227)
        at org.apache.sqoop.manager.ConnManager.getColumnTypes(ConnManager.java:295)
        at org.apache.sqoop.orm.ClassWriter.getColumnTypes(ClassWriter.java:1833)
        at org.apache.sqoop.orm.ClassWriter.generate(ClassWriter.java:1645)
        at org.apache.sqoop.tool.CodeGenTool.generateORM(CodeGenTool.java:107)
        at org.apache.sqoop.tool.ImportTool.importTable(ImportTool.java:478)
        at org.apache.sqoop.tool.ImportTool.run(ImportTool.java:605)
        at org.apache.sqoop.Sqoop.run(Sqoop.java:143)
        at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
        at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:179)
        at org.apache.sqoop.Sqoop.runTool(Sqoop.java:218)
        at org.apache.sqoop.Sqoop.runTool(Sqoop.java:227)
        at org.apache.sqoop.Sqoop.main(Sqoop.java:236)
16/11/28 14:30:27 ERROR tool.ImportTool: Encountered IOException running import job: java.io.IOException: No columns to generate for ClassWriter
        at org.apache.sqoop.orm.ClassWriter.generate(ClassWriter.java:1651)
        at org.apache.sqoop.tool.CodeGenTool.generateORM(CodeGenTool.java:107)
        at org.apache.sqoop.tool.ImportTool.importTable(ImportTool.java:478)
        at org.apache.sqoop.tool.ImportTool.run(ImportTool.java:605)
        at org.apache.sqoop.Sqoop.run(Sqoop.java:143)
        at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
        at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:179)
        at org.apache.sqoop.Sqoop.runTool(Sqoop.java:218)
        at org.apache.sqoop.Sqoop.runTool(Sqoop.java:227)
        at org.apache.sqoop.Sqoop.main(Sqoop.java:236)
```

解决办法：增加参数`--driver com.mysql.jdbc.Driver`。

`PS: 另需注意mysql java驱动包的版本为5.1.31以上。`


# No primary key could be found

遇到错误如下：

```
16/11/28 14:39:03 ERROR tool.ImportTool: Error during import: No primary key could be found for table czw_test_distinct. Please specify one with --split-by or perform a sequential import with '-m 1'.
```

原因：mysql数据库的表中需要有个主键，如果没有主键的话需要手动选取一个合适的拆分字段，如下:
`bin/sqoop import --connect jdbc:mysql://10.10.10.84:3306/test --table czw_test_distinct --username ---------- --password ---------- --target-dir /tmp/testSqoop --driver com.mysql.jdbc.Driver --split-by a`

`注意：主键空的数据无法导入`，如：
原表：
a	b	c
a1		
\N	b1	\N
\N	\N	c1
a1	b1	c1
导入后：
a1,,
a1,b1,c1


