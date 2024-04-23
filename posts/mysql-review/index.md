# MySQL 优化及索引设计规范

索引是帮助MySQL高效获取数据的排好序的数据结构。

## 几个MySQL命令

```shell
mysql.server start // 启动MySQL
mysql.server stop  // 停止MYSQL
./mysqld_safe --data=../data // 从data备份中恢复数据
./mysql_secure_installation // 修改管理员密码
```
&lt;!--more--&gt;

## 索引的优缺点

优点

- 索引大大减少了服务器需要扫描的数据量。
- 索引可以帮助服务器避免排序和临时表。
- 索引可以将随机IO变为顺序IO。

缺点


- 创建索引和维护索引要耗费时间 ，这种时间随着数据量的增加而增加。
- 索引需要占物理空间 。

## 索引的分类

- 主键索引(聚簇索引、聚集索引)、二级索引（非聚集索引）。
- 普通索引、唯一索引、全文索引。 
- 独立索引、复合索引、前缀索引。
- B&#43;Tree 索引、Hash索引。

## 索引数据结构

常见的索引数据结构有Hash表、二叉树、平衡二叉树、红黑树、B-Tree、B&#43;Tree。
![](/_static/mysql/m0.png)

1. Hash 索引：Hash 表只能做等值匹配，效率很高。但是不支持范围查找和排序，因为取每个数据要做hash运算，只有取出来才能知道他是什么。
2. 二叉树：二叉树极端情况下树会变成一个链表，也不适合做索引。
3. 平衡二叉树：平衡二叉树无法支持很大的数据，数据量大时，树的高度依然很高，效率不高。
4. 红黑树：红黑树通过限制节点的颜色控制数据变化时树的旋转次数，插入、删除数据时性能有一定提升，但树高依然无法控制，无法支持大数据量。
5. B-tree：非叶子节点存储真实数据，占用空间大，可存储的索引会减少。
6. B&#43;tree：所有数据存在叶子节点，非叶子节点只存索引，可容纳更多的索引数据。 SQL性能分析工具Explain 工具详解 在一条查询语句前加 explain
   可以获得SQL语句的执行计划，可看到使用了什么索引，大致扫描了多少行等信息，从而分析出SQL语句的瓶颈在哪里。 explain select * from employees where name = &#39;Lucy&#39;;

## Explain 工具详解

在一条查询语句前加 explain 可以获得SQL语句的执行计划，可看到使用了什么索引，大致扫描了多少行等信息，从而分析出SQL语句的瓶颈在哪里。

```sql
explain select * from employees where name = &#39;Lucy&#39;;
```

![](/_static/mysql/m1.png)
下面介绍下 Explain 中的列。

- id

一条SQL语句在MySQL底层可要分成条语句执行，比如关联查询，ID代表SQL语句执行的优先级，ID越大的SQL先执行，ID一样的从上到下一次执行，ID为NULL最后执行。

- select_type select_type

代表要执行的SQL语句复杂还是简单，取值有如下几种。 simple : 简单查下，不包括子查询或uniun。 primary：代表复杂查询总最外层的select。 subquery：包含在 select 中的子查询（不在 from
子句中）。 derived：包含在from语句中的子查询，代表用到临时表。 union：包含union 查询。

- table

代表正在查询的是哪个表，table 如果是 &lt;derivenN&gt;，表示使用的临时表，N 就是依赖查询的ID的值，会先执行id=N 语句。当有union 时，table 列为&lt;union1,2&gt;,1,2 分别代表参与SQL的ID。

- type

表示关联类型或访问类型, 查找数据的大致范围，性能从最优到最差一次为：system &gt; const &gt; eq_ref &gt; ref &gt; range &gt; index &gt; ALL，一般DBA要求要达到range，最好是ref。
const或system：primary key 或 unique key 的所有列与常数比较，只匹配一行，速度很快。 eq_ref：primary key 或 unique key 索引的所有部分被连接使用，只返回一行。
ref：普通索引或唯一索引前缀匹配，可能找到多行。 range：范围查找，如：in、between、&gt; 、&lt;、&gt;= 等。 index：扫描二级索引的全部叶节点，一般为覆盖索引。 ALL：扫描主键索引的叶节点，指全表扫描。

- possible_keys

代表MySQL可能选择哪些索引来查找。 当possible_keys有值，key 为NULL时，代表数据不多，走索引还要回表，MySQL 认为直接走全表扫描可能会快一点；
当possible_keys为NULL时，代表没有索引可用，直接全表扫描。

- key

代表查询实际使用的索引，为NULL表示没有选用索引，如果有可用的索引可以用 force index强制走索引，ignore index忽略索引。 select * from employees force index(
idx_name_age_position) where name = &#39;Lucy&#39;;

- key_len

代表查询使用索引的字节数，通过这个值可以算出具体使用索引中的那一列，比如当索引是联合索引的时候。 key_len计算规则如下： 字符串，char(n)和varchar(n)
，5.0.3以后版本中，n均代表字符数，而不是字节数，如果是utf-8，一个数字或字母占1个字节，一个汉字占3个字节。 char(n)：如果存汉字长度就是 3n 字节 varchar(n)：如果存汉字则长度是 3n &#43; 2
字节，加的2字节用来存储字符串长度，因为varchar是变长字符串 数值类型 tinyint：1字节 smallint：2字节 int：4字节 bigint：8字节 时间类型 date：3字节timestamp：4字节
datetime：8字节 如果字段允许为 NULL，需要1字节记录是否为 NULL 索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。

- ref

代表key对于的索引中，查找值用到的列或常量，常见的有 const、字段名。

- rows

代表大概扫描的行数，不是结果集准确的行数。

- Extra 
  

代表额外的信息,取值如下:

    - Using index：使用覆盖索引。
    - Using where：查询的列没有索引。
    - Using index condition：查询的列表未完全被索引覆盖，要加索引优化。
    - Using temporay：需要临时表处理查询，要优化。 
    - Using filesort：借用外部空间排序，数据少用内存，数据大用磁盘，要加索引优化。
    - Select tables optimized away：使用聚合函数访问索引字段，如：max,min 等。 

## 索引最佳实战

实例表

```sql
CREATE TABLE `employees`
(
    `id`        int(11) NOT NULL AUTO_INCREMENT,
    `name`      varchar(24) NOT NULL DEFAULT &#39;&#39; COMMENT &#39;姓名&#39;,
    `age`       int(11) NOT NULL DEFAULT &#39;0&#39; COMMENT &#39;年龄&#39;,
    `position`  varchar(20) NOT NULL DEFAULT &#39;&#39; COMMENT &#39;职位&#39;,
    `hire_time` timestamp   NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT &#39;入职时间&#39;,
    PRIMARY KEY (`id`),
    KEY         `idx_name_age_position` (`name`,`age`,`position`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 COMMENT=&#39;员工记录表&#39;;

```

常见实例

- 不在索引上做任何操作，如：计算、函数、类型转换等，会导致索引失效。
```sql
EXPLAIN SELECT * FROM employees WHERE left (name, 3) = &#39;LiLei&#39;;
```

- 存储引擎不能使用索引范围条件右边的列。如下只有 nage 和 age 字段参与索引查找。可通过 key_len 长度确定索引真实用到的字段。
```sql
EXPLAIN SELECT * FROM employees WHERE name = &#39;LiLei&#39; AND age &gt; 22 AND position = &#39;manager&#39;;
```

- 尽量用覆盖索引，减少SLECT * 语句。
```sql
EXPLAIN SELECT name, age FROM employees WHERE name = &#39;LiLei&#39; AND age = 23 AND position = &#39;manager&#39;;
```

- 不等于(!=或者&lt;&gt;)、not in、not exists 都不走索引。
```sql
EXPLAIN SELECT * FROM employees WHERE name != &#39;LiLei&#39;;
```

- &lt;、&gt;、&lt;=、&gt;= 索引优化器会根据检索比例、表大小等因素评估是否走索引。
```sql
EXPLAIN SELECT * FROM employees WHERE name &gt;= &#39;LiLei&#39;;
```

- is null、is not null 一般也不走索引。
```sql
EXPLAIN SELECT * FROM employees WHERE name is null
```

- like 查询通配符开都不走索引。
```sql
EXPLAIN SELECT * FROM employees WHERE name like &#39;%Lei&#39;
```

- 字符串不加单引号会使索引失效。
```sql
EXPLAIN SELECT * FROM employees WHERE name = 1000;
```

- 少用 or 或 in, 查询时不一定走索引，优化器评估确定是否走索引。
```sql
EXPLAIN SELECT * FROM employees WHERE name = &#39;LiLei&#39; or name = &#39;HanMeimei&#39;;
```

- 强制走索引
```sql
EXPLAIN SELECT * FROM employees force index(idx_name_age_position)
WHERE name &gt; &#39;LiLei&#39; AND age = 22 A ND position =&#39;manager&#39;;
```
虽然使用了强制走索引让联合索引第一个字段范围查找也走索引，扫描的行rows看上去也少了点，但是最终查找效率不一定比全表 扫描高，因为回表效率不高。

- 索引下推
```sql
EXPLAIN SELECT * FROM employees WHERE name like &#39;LiLei%&#39; AND age = 22 AND position = &#39;manager&#39;;
```

在MySQL5.6之前的版本，这个查询只能在联合索引里匹配到名字是&#39;LiLei&#39;开头的索引，然后拿这些索引对应的主键逐个回表，到主键索引上找出相应的记录，再比对age和position这两个字段的值是否符合。

MySQL 5.6引入了索引下推优化，可以在索引遍历过程中，对索引中包含的所有字段先做判断，过滤掉不符合条件的记录之后再回表，可以有效的减少回表次数。使用了索引下推优化后，上面那个查询在联合索引里匹配到名字是&#39;LiLei&#39;
开头的索引之后，同时还会在索引里过滤age和position这两个字段，拿着过滤完剩下的索引对应的主键id再回表查整行数据。

索引下推会减少回表次数，对于innodb引擎的表索引下推只能用于二级索引，innodb的主键索引（聚簇索引）树叶子节点上保存的是全行数据，所以这个时候索引下推并不会起到减少查询全行数据的效果。

&lt;font color=&#34;red&#34;&gt;为什么范围查找Mysql没有用索引下推优化？&lt;/font&gt;

估计应该是Mysql认为范围查找过滤的结果集过大，likeKK%在绝大多数情况来看，过滤后的结果集比较小，所以这里Mysql选择给likeKK%用了索引下推优化，当然这也不是绝对的，有时like
KK% 也不一定就会走索引下推。

## 一条SQL语句是如何执行的

大体来说，MySQL 可以分为 Server 层和存储引擎层两部分。
![](/_static/mysql/m2.png)

- Server层

主要包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

- Store层存

储引擎层负责数据的存储和提取。其架构模式是插件式的，支持InnoDB、MyISAM、Memory等多个存储引擎。现在最常用的存储引擎是InnoDB，它从MySQL 5.5.5
版本开始成为了默认存储引擎。也就是说如果我们在createtable时不指定表的存储引擎类型,默认会给你设置存储引擎为InnoDB。

- 连接器

负责与各种客户端建立连接、获取权限、维持和管理连接。
连接过程可分为两步：建立连接，然后要求输入用户名密码认证身份，认证成功后会获取用户所有权限放在用户的Session中。（如果此时修改了系统表user中的权限，也不影响已经建立了的连接，除非断开重连才会生效）。

查看已连接的用户：
```sql
show processlist; -- Commend为Sleep为空闲连接。
```

查看连接默认时长：
```sql
show global variables like &#34;wait_timeout&#34;;  -- 默认值8小时。
```

修改连接默认时长：
```sql 
global wait_timeout=28800;  -- 设置全局服务器关闭非交互连接之前等待活动的秒数
```

- 查询缓存

MySQL收到查询语句时，首先会到查询缓存查询，查不到再去查库，缓存的Key为SQL语句，Value就是查询结果。 只要对一个表更新，这个表上的所有查询缓存就会被清空。一般可以用在静态表中，对于频繁更新的表意义不大。

开启查询缓存：
```text
修改my.cnf query_cache_type=2  // 0代表关闭查询缓存OFF，1代表开启ON，2（DEMAND按需，默认都不走缓存）
```

指定走查询缓存：
```sql
select SQL_CACHE * from test where ID = 5; -- 用 SQL_CACHE 显示指定走缓存。
```

查看缓存是否开启：
```sql
show global variables like &#34;%query_cache_type%&#34;;
```

控缓存命中率：
```sql
show status like&#39;%Qcache%&#39;;
```
mysql8.0已经移除了查询缓存功能。

- 分析器

首先此法分析器会分析每个词代表什么，然后会做语法分析，如果你的语句不对，就会收到“You have an error in your SQL syntax”的错误提醒，经过此法分析器后生成一颗语法树。

- 优化器

优化SQL语句，选择走哪一个索引，或是全表扫码。

- 执行器

先做一次权限校验，再调用InnoDB的接口执行查询，获取结果；

Bin-log 日志 记录CUD执行的日志，用于恢复数据。

## MySQL是如何选择索引的 (Trace 工具)

Trace 工具可以清楚的看到一条SQL的详细执行步骤，开启 Trace 工具对MySQL性能有一定的影响，一般只有临时分析问题时开启，用完关闭。

开启

```sql
mysql
&gt; set session optimizer_trace=&#34;enabled=on&#34;,end_markers_in_json=on; -- 开启trace
```

实例

```sql
SELECT * FROM employees where name &gt; &#39;a&#39; order by position;
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

先执行一条查查语句，紧接着执行上面第二条语句，就能获得SQL执行的 Trace 计划，如下为关键信息说明。 Trace
结果说明文档参考：http://note.youdao.com/noteshare?id=d2e8a0ae8c9dc2a45c799b771a5899f6
结论：全表扫描的成本低于索引扫描，所以mysql最终选择全表扫描。

```sql
mysql
&gt; set session optimizer_trace=&#34;enabled=off&#34;; --关闭trace
```

## MySQL 深入优化之 Order by 与 Group By 优化

```sql
explain select * from employees where name = &#39;LiLei&#39; order by age;
```

```json
[
  {
    &#34;id&#34;: 1,
    &#34;select_type&#34;: &#34;SIMPLE&#34;,
    &#34;table&#34;: &#34;employees&#34;,
    &#34;type&#34;: &#34;ref&#34;,
    &#34;possible_keys&#34;: &#34;idx_name_age_position&#34;,
    &#34;key&#34;: &#34;idx_name_age_position&#34;,
    &#34;key_len&#34;: &#34;74&#34;,
    &#34;ref&#34;: &#34;const&#34;,
    &#34;rows&#34;: 1,
    &#34;Extra&#34;: &#34;Using index condition; Using where&#34;
  }
]
```
Extra 中没有 Using fileSort, 使用name索引字段，紧跟着age排序。

```sql
explain select * from employees where name = &#39;LiLei&#39; order by position;
```

```json
[
  {
    &#34;id&#34;: 1,
    &#34;select_type&#34;: &#34;SIMPLE&#34;,
    &#34;table&#34;: &#34;employees&#34;,
    &#34;type&#34;: &#34;ref&#34;,
    &#34;possible_keys&#34;: &#34;idx_name_age_position&#34;,
    &#34;key&#34;: &#34;idx_name_age_position&#34;,
    &#34;key_len&#34;: &#34;74&#34;,
    &#34;ref&#34;: &#34;const&#34;,
    &#34;rows&#34;: 1,
    &#34;Extra&#34;: &#34;Using index condition; Using where; Using filesort&#34;
  }
]
```

Extra 中出现 Using fileSort, 使用name索引字段，position排序，中间断了。

1、MySQL支持两种方式的排序filesort和index，Using index是指MySQL扫描索引本身完成排序。index效率高，filesort效率低。

2、order by满足两种情况会使用Using index。
 1) order by语句使用索引最左前列。
 2) 使用where子句与order by子句条件列组合满足索引最左前列。
    

3、尽量在索引列上完成排序，遵循索引建立（索引创建的顺序）时的最左前缀法则。

4、如果order by的条件不在索引列上，就会产生Using filesort。

5、能用覆盖索引尽量用覆盖索引 

6、group by与order by很类似，其实质是先排序后分组，遵照索引创建顺序的最左前缀法则。对于group by的优化如果不需要排序的可以加上order by
   null禁止排序。注意，where高于having，能写在where中 的限定条件就不要去having限定了。

Using filesort文件排序原理 

- 单路排序： 是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序；用trace工具可以看到sort_mode信息里显示&lt; sort_key,
additional_fields &gt;或者&lt;sort_key,packed_additional_fields &gt;

- 双路排序（又叫回表排序模式）： 是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行 ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段；用trace工具可以看到sort_mode信息里显示&lt;
sort_key, rowid &gt;

MySQL 通过比较系统变量 max_length_for_sort_data(默认1024字节) 的大小和需要查询的字段总大小来判断使用哪种排序模式。
    
    1、如果 字段的总长度小于max_length_for_sort_data ，那么使用 单路排序模式； 
    2、如果 字段的总长度大于max_length_for_sort_data ，那么使用 双路排序模∙式。

## MySQL 深入优化之分页优化

创建实例表：

```sql
employees
CREATE TABLE `employees`
(
    `id`        int(11) NOT NULL AUTO_INCREMENT,
    `name`      varchar(24) NOT NULL DEFAULT &#39;&#39; COMMENT &#39;姓名&#39;,
    `age`       int(11) NOT NULL DEFAULT &#39;0&#39; COMMENT &#39;年龄&#39;,
    `position`  varchar(20) NOT NULL DEFAULT &#39;&#39; COMMENT &#39;职位&#39;,
    `hire_time` timestamp   NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT &#39;入职时间&#39;,
    PRIMARY KEY (`id`),
    KEY         `idx_name_age_position` (`name`,`age`,`position`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT=&#39;员工记录表&#39;;

```

- 场景一：主键自增且连续的分页

```sql
EXPLAIN select * from employees limit 10000,10;
```

一般会这么写SQL，Mysql 会先取10010条数据，然后抛弃前面10000条返回剩下的10条，当表数据量越来越大是查询会越来越慢。 优化后： mysql&gt; EXPLAIN select * from employees where
id &gt; 90000 limit 5;

此种方式会走索引，且扫描行数大大减少，执行效率很高，但要求苛刻，必须有自增ID且连续，不能断（删数据），一般很难满足，不常用。

- 场景二：非主键字段排序的分页

```sql
EXPLAIN select * from employees ORDER BY name limit 90000,5;
```

即使 name 字段有索引，MySQL 也不会走索引，原因：走 idx_name_age_position 索引，还要回表，要扫描多棵树，MySQL 认为性能不高，还不如全部扫描呢，所以还用了 filesort 排序。

优化后:

```sql
EXPLAIN select *from employees e
inner join (select id from employees order by name limit 90000,5) ed on e.id = ed.id;
```

按照ID优先级 Mysql 的执行顺序为 2:1:1（ID越大先执行，ID一样从上到下执行）, 先走idx_name_age_position
覆盖索引，找到所有满足条件的记录ID列表，再拿这个结果去聚簇索引扫描。这里用到了临时表 &lt;derived2&gt;,type 虽然为ALL，表示全表扫描，但这个临时表也只有5条记录，所以很快。
优化关键：想办法让条件查询、排序返回的值尽量最少，尽量走索引，再取那这个结果到主键索引中查。

## MySQL深入优化之 关联查询 Join 优化

创建实例表 t1,t2

```sql
CREATE TABLE `t1`
(
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `a`  int(11) DEFAULT NULL,
    `b`  int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY  `idx_a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

创建表t2，和t1一样

```sql
create table t2 like t1;
```
然后：t1 表插入1w条记录，t2表插入100条记录； t1 大表，t2小表；同时两个表 a 字段都都有索引。

**Join 查询有两种算法 Nested-Loop Join 算法（嵌套循环连接） Block Nested-Loop Join 算法（基于块的嵌套循环连接）**

- 第一种：嵌套循环连接 Nested-Loop Join(NLJ) 算法

```sql
EXPLAIN select * from t1 inner join t2 on t1.a = t2.a;
```
![](/_static/mysql/m5.png)

执行计划原理图
![](/_static/mysql/m3.png)

执行步骤

- 从t2表中读取一条记录。
- 从第一步的数据中取出关联字段a，到t1中查找。 
- 取出t1中满足的行和t2中获取的结果集合并。 
- 重复以上3步，直达t2中的记录取完，终止返回。

执行过程分析

t1 表全表扫描，扫描100行，t2表根据索引a扫描二级索引，一次可以扫描t1的一整行完整数据，扫描了100行，因此总共扫描了200行。这种方式称为NLJ算法扫描，特征就是 Extra 中没有 Using join
buffer 内容。 那么如果 a 字段没有索引呢，那就走下面的 BNL算法。

- 第二种：基于块的嵌套循环连接 Block Nested-Loop Join(BNL)算法

```sql
EXPLAIN select * from t1 inner join t2 on t1.b = t2.b;
```
![](/_static/mysql/m5.png)
明显 Extra 中有Using join buffer。。。说明就是BNL算法，因为关联字段 b 上没索引。

BNL 执行计划
![](/_static/mysql/m4.png)

执行步骤 

- 把 t2 的所有数据放入到 join_buffer 中。 
- 把表 t1 中每一行取出来，跟 join_buffer 中的数据做对比。 
- 返回满足 join 条件的数据。

执行过程分析

join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果放不下表 t2 的所有数据话，策略很简单，就是分段放。整个过程对表 t1 和 t2
都做了一次全表扫描，因此扫描的总行数为10000(表 t1 的数据总量) &#43; 100(表 t2 的数据总量) =10100。并且 join_buffer 里的数据是无序的，因此对表 t1 中的每一行，都要做 100
次判断，所以内存中的判断次数是100 * 10000= 100 万次。

关联SQL的优化原则

关联字段加索引，让mysql做join操作时尽量选择NLJ算法。 小表驱动大表，写多表连接sql时如果明确知道哪张表是小表可以用straight_join写法固定连接驱动方式，省去mysql优化器自己判断的时间。

## MySQL 深入优化之 COUNT(*) 优化

实例SQL

```sql
EXPLAIN select count(1) from employees;
```
```sql
EXPLAIN select count(id) from employees;
```
```sql
EXPLAIN select count(name) from employees;
```
```sql
EXPLAIN select count(*) from employees;
```

执行效率结论
- 字段有索引：count(*)≈count(1)&gt;count(字段)&gt;count(主键 id)
- 字段无索引：count(*)≈count(1)&gt;count(主键 id)&gt;count(字段)

执行计划分析
- count(字段)：如果字段有索引，走二级索引，每扫描一行取出字段的值，统计字段的个数,NULL值不算。 
- count(1)：如果有二级索引，扫描二级索引，累加字段用常量1代替，不需要取出字段的值。 
- count(*)：如果有二级索引，走二级索引，不需要取出字段的值，扫描到行就会统计加1。 
- count(id)：如果有二级索引，走二级索引,没有才会走主键索引，因为二级索引数据少，扫描快一些。

优化方案 

单独维护总行数。myisam 存储引擎的表，行数会维护在磁盘上，count时会直接返回，不再扫描。 show table status like &#39;表名&#39;， 可得到表的预估行数。 将行数维护在 redis 里，但一致性很难保证。
将行数维护在本地库，插入和删除和更新行数放在一个事务里，可保证一致性。 注意：count(字段)不会计算null值的行。

## MySQL 深入优化之 如何选择数据类型

- 选择数据类型的原则
    - 确定合适的大类型：数字、字符串、时间、二进制；
    - 确定具体的类型：有无符号、取值范围、变长定长等。
- 注意事项
    - int(11) 和 int(5)：数字代表显示宽度，没有实际意义。如果类型后面增加 ZEROFILL，则宽度不足时前面会用0补齐。
    - char(5)：存储固定长度字符串，5代表字符长度，如果不够5个前面会用空格代替，占用存储空间是固定的。char最多255字节。
    - varchar(20)：20代表最多存储字符的长度。varchar 最长 65535 个字节。

## 索引优化建议及设计原则

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/mysql-review/  

