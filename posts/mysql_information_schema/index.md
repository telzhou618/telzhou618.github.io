# MYSQL 表空间分析


## 删除数据并释放空间命令
1. 立刻释放磁盘空间 ，不管是 Innodb和MyISAM
```sql
drop table table_name 
```
2. 立刻释放磁盘空间 ，不管是 Innodb和MyISAM
```sql
truncate table table_name
```
3. 删除表的全部数据，对于MyISAM 会立刻释放磁盘空间 ，而InnoDB 不会释放磁盘空间
```sql
delete from table_name 
```
4. 带条件的删除, 不管是innodb还是MyISAM都不会释放磁盘空间；delete操作后使用optimize table table_name 释放磁盘空间，优化表期间会锁定表，所以要在空闲时段执行optimize table
```sql
delete from table_name where xx 
```

&lt;!--more--&gt;

## 查看表空间占用大小
MySQL  表空间信息保存在哪 information_schema.TABLES  中。

- 查看所有数据库的容量和大小
```sql
select
table_schema as &#39;数据库&#39;,
sum(table_rows) as &#39;记录数&#39;,
sum(truncate(data_length/1024/1024, 2)) as &#39;数据容量(MB)&#39;,
sum(truncate(index_length/1024/1024, 2)) as &#39;索引容量(MB)&#39;,
sum(truncate(data_free/1024/1024, 2)) as &#39;碎片空间容量(MB)&#39;
from information_schema.tables
group by table_schema
order by sum(data_length) desc, sum(index_length) desc
```

- 查看指定数据库各个表容量大小
```sql
select
table_schema as &#39;数据库&#39;,
table_name as &#39;表名&#39;,
table_rows as &#39;记录数&#39;,
truncate(data_length/1024/1024, 2) as &#39;数据容量(MB)&#39;,
truncate(index_length/1024/1024, 2) as &#39;索引容量(MB)&#39;,
sum(truncate(data_free/1024/1024, 2)) as &#39;碎片空间容量(MB)&#39;
from information_schema.tables
where table_schema=&#39;mysql&#39; -- 数据库名称
order by data_length desc, index_length desc;
```

- 查询所有数据库占用磁盘空间大小
```sql
select 
TABLE_SCHEMA, 
concat(truncate(sum(data_length)/1024/1024,2),&#39; MB&#39;) as data_size,
concat(truncate(sum(index_length)/1024/1024,2),&#39; MB&#39;) as index_size,
concat(truncate(sum(data_free)/1024/1024,2),&#39; MB&#39;) as data_free_size
from information_schema.tables
group by TABLE_SCHEMA
ORDER BY data_size desc;
```

- 查询单个库中所有表磁盘空间大小
```sql
select 
TABLE_NAME, 
concat(truncate(data_length/1024/1024,2),&#39; MB&#39;) as data_size,
concat(truncate(index_length/1024/1024,2),&#39; MB&#39;) as index_size,
concat(truncate(sum(data_free)/1024/1024,2),&#39; MB&#39;) as data_free_size
from information_schema.tables 
where TABLE_SCHEMA = &#39;查询的表名&#39;
group by TABLE_NAME
order by data_length desc;
```
&gt; truncate 是MYSQL的系统函数，作用是按照小数点截取，但不进行四舍五入， TRUNCATE(X,D) ，其中X是数值，D是保留小数的位数。
&gt; 如： TRUNCATE(123.4567, 3); 结果是 123.456，TRUNCATE(123.4567, 2); 结果是 123.45

- information_schema.TABLES   表常用字段及说明

  | 字段 | 含义 |
  | --- | --- |
  | Table_catalog | 数据表登记目录 |
  | Table_schema | 数据表所属的数据库名 |
  | Table_name | 表名称 |
  | Table_type | 表类型[system view&amp;#124;base table] |
  | Engine | 使用的数据库引擎[MyISAM&amp;#124;CSV&amp;#124;InnoDB] |
  | Version | 版本，默认值10 |
  | Row_format | 行格式[Compact&amp;#124;Dynamic&amp;#124;Fixed] |
  | Table_rows | 表里所存多少行数据 |
  | Avg_row_length | 平均行长度 |
  | Data_length | 数据长度 |
  | Max_data_length | 最大数据长度 |
  | Index_length | 索引长度 |
  | Data_free | 空间碎片 |
  | Auto_increment | 做自增主键的自动增量当前值 |
  | Create_time | 表的创建时间 |
  | Update_time | 表的更新时间 |
  | Check_time | 表的检查时间 |
  | Table_collation | 表的字符校验编码集 |
  | Checksum | 校验和 |
  | Create_options | 创建选项 |
  | Table_comment | 表的注释、备注 |

## 参考

- MYSQL 删除数据后释放空间：[https://www.jianshu.com/p/ebe6ac68099a](https://www.jianshu.com/p/ebe6ac68099a)
- MySQL 查看数据库表容量大小和磁盘空间占用大小： [https://my.oschina.net/90design/blog/4330825](https://my.oschina.net/90design/blog/4330825)
- mysql中information_schema.tables字段说明：[https://blog.csdn.net/weixin_40918067/article/details/116868906](https://blog.csdn.net/weixin_40918067/article/details/116868906)
- MySQL的TRUNCATE()函数： [https://blog.csdn.net/Fekerkk/article/details/122536574](https://blog.csdn.net/Fekerkk/article/details/122536574)


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/mysql_information_schema/  

