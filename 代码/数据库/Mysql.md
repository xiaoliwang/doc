# MySQL 数据库

[TOC]

## 建表规范

### 一般

1. 【强制】表名、字段名必须使用**小写字母**， **数字**或**下划线**。

   禁止：出现数字或下划线开头，`1_biao`，`_biao`

   禁止：两个下划线中间只出现数字，`level_3_name`

2. 【强制】表名、字段名禁用保留字，如 **desc**，**order**等，情参考 [MySQL 官方保留字](https://dev.mysql.com/doc/refman/5.7/en/keywords.html)

3. 【推荐】库名与应用名称尽量一致。

4. 【推荐】单表行数超过 **500** 万行或者单表容量超过 **2GB** ， 才推荐进行分库分表。

5. 【推荐】如果修改字段含义或对字段表示的状态追加时，需要及时更新字段注释。




### 表名

1. 【强制】表名不适用复数名词
2. 【推荐】表名最好是加上“业务名称_表的作用”。




### 字段

1. 【强制】任何字段如果为非负数，必须是`unsigned`
2. 【强制】 表达**是与否**概念的字段，必须使用`is_xxx`的方式命名，数据类型是 `unsigned tinyint`
3. 【强制】定点数类型使用 `decimal` 和 `int` 类型，禁止在一般情况下使用 `float` 和 `double` 类型。如果数据范围超过 `decimal` 的范围，建议将数据拆成 **整数** 和 **小数** 两部分。
4. 【强制】如果存储的字符串长度几乎相等，使用 `char` 定长字符串类型
5. 【强制】`varchar` 是可变长字符串，长度不要超过 **5000**， 如果存储长度大于 **5000** ，定义字段类型为 `text` ，独立出来一张表，用主键来对应，避免影响其它字段索引效率。
6. 【推荐】字段允许适当冗余，以提高查询性能，但必须考虑数据一致。冗余字段应遵循：
   1. 不是频繁修改的字段
   2. 不是 `varchar` 超长字段，更不能是 `text` 字段




### 索引

1. 【强制】主键索引名为`pk_字段名`；唯一索引名为`uk_字段名`; 普通索引名则为`idx_字段名`。

2. 【强制】表必备三字段：`id`，`create_time `，`modified_time`。

   1. `id` 为主键，类型为 `unsigned bigint` 、单表时自增、步长为1
   2. `create_time` 为字段创建时间，类型为 `unsigned int`
   3. `modified_time` 为字段修改时间，类型为 `unsigned int`

   ​


## 索引规范

1. 【强制】 业务上具有**唯一**特性的字段，即使是多个字段组合，也必须建成**唯一索引**。

   注：唯一索引对 insert 的速度损耗可以忽略。

2. 【强制】禁止使用三个表的连查（join）。需要 join 的字段，数据类型必须绝对一致；多表关联查询时，保证被关联的字段需要有索引

3. 【强制】在 **varchar** 字段上建立索引时，必须指定 **索引长度** ，没必要对全字段建立索引，根据实际文本区分度决定索引长度即可。一般长度为 20 的索引，区分度会高达 90%。

   ```sql
   -- 查看区分度
   count(distinct left(coloumn, index_length))/count(*)
   ```

4. 【强制】页面搜索严禁 **左模糊** 或者 **全模糊**， 如果需要请使用搜索引擎。

   ```sql
   -- 不能使用下面两种查询
   select * from table where column1 like "%xxx"
   select * from table where column1 like "%xxx%"
   ```

5. 【推荐】如果有 order by 的场景，请注意利用索引的 **有序性** 。order by 最后的字段是组合索引的一部分，并且放在索引组合顺序的最后。

   ```sql
   -- 利用有序性例子
   where a = ? and b = ? order by c -- index 设置为 idx_a_b_c
   -- 无法利用有序性例子
   where a > 10 order by b
   ```

   ​

6. 【推荐】利用 **覆盖索引** 来进行查询操作，避免回表。

   ```sql
   explain sql -- extra 列会出现 using index
   ```

7. 【推荐】利用延迟关联或者子查询优化 offset 很大的分页场景。

   说明：MySQL 使用 offset 时，是获取 offset + N 行，放弃前 offset 行，返回 N 行。所以效率较低。

   ```sql
   select a.* from table1 a, (select id from table1 where condition limit 100000, 20) b where a.id = b.id
   ```

8. 【推荐】SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，如果可以是 consts  最好。（这里是指 explain 时 type 的值）
   1. consts 但表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可取到数据。
   2. ref 指的是使用普通索引 (normal index)。
   3. range 对索引进行范围索引

9. 【推荐】建组合索引的时候，区分度最高的在左边。

   ```sql
   -- a列的几乎接近于唯一值，只需要单建 idx_a 索引即可。
   select * from table where a = ? and b = ?;
   -- 如果存在非等号判断时，索引需把等号条件的列前置。
   select * from table where a > ? and b = ?; -- 该例必须把 b放在索引的最前列
   ```

10. 【推荐】防止因字段类型不同造成的隐式转换，导致索引失效。

  ```sql
  -- 禁止出现以下语句
  select * from table where a = "xxx" and a = xxx;
  ```

11. 【参考】创建索引时避免有如下极端误解：
    1. 宁滥勿缺。 误认为一个查询就需要建一个索引。
    2. 宁缺勿滥。 误认为索引会消耗空间、严重拖慢更新和新增速度。
    3. 抵制惟一索引。 误认为业务的惟一性一律需要在应用层通过“先查后插”方式解决。 




## SQL

1. 【**强制**】使用 **count(*)** ，count(*) 是 SQL92 定义的标准统计行数语法

2. 【**强制**】**count(distinct col)** 计算该列除 NULL 之外的不重复行数

  ```sql
  -- 例子：如果其中一个 column2 全为 NULL 时，即使 column1 全是不同的值
  select count(distinct col1, col2) -- 返回 0
  ```

3. 【**强制**】当某一列的值全是 NULL 时，count(col) = 0; sum(col) = NULL, 使用 **sum(col)** 注意 **NPE** 问题。 
   ```sql
   -- 正例
   select if(ISNULL(sum(col)), 0 SUM(col)) from table;
   ```

4. 【**强制**】使用 ISNULL()来判断是否为 NULL 值。注意： NULL 与任何值的直接比较都为 NULL。

   ```sql
   NULL <> NULL -- NULL
   NULL = NULL -- NULL
   NULL <> 1 -- NULL
   ```

5. 【**强制**】在代码中写分页查询逻辑时，若 count 为 0 应直接返回，避免执行后面的分页语句。 

6. 【**强制**】不得使用外键与级联，一切外键概念必须在应用层解决。

   说明： 

   主键和外键：学生表中的 student_id 是主键，那么成绩表中的 student_id 则为外键。
   级联更新：如果更新学生表中的 student_id，同时触发成绩表中的 student_id 更新，则为级联更新。
   原因：

   外键与级联更新适用于单机低并发，不适合分布式、高并发集群； 级联更新是强阻塞，存在数
   据库更新风暴的风险； 外键影响数据库的插入速度。 

7. 【**强制**】禁止使用存储过程，存储过程难以调试和扩展，更没有移植性。

8. 【**强制**】数据订正时，删除和修改记录时，要先 select，避免出现误删除，确认无误才能执
   行更新语句。

9. 【**特别强制**】查询语句至少要达到 range 级别，要求是 ref 级别，如果可以是 consts  最好。（这里是指 explain 查询语句时的 type 值）

10. 【推荐】 in 操作能避免则避免，若实在避免不了，需要仔细评估 in 后边的集合元素数量，控
  制在 1000 个之内 。

11. 【参考】 如果有全球化需要，使用字符集 `utf-8` 编码。并使用下列两种排序规则：

    1. 不区分大小写：`utf8_general_ci`
    2. 区分大小写：`utf8_bin`

    注意字符统计函数

    ```sql
    SELECT LENGTH("轻松工作"); -- 返回 12
    SELECT CHARACTER_LENGTH("轻松工作"); -- 返回 4
    ```

    如果要使用表情，那么使用 `utfmb4` 来进行存储，注意它与 `utf-8` 编码的区别。

    如果需要区分大小写查询，清

12. 【参考】 `TRUNCATE TABLE` 比 `DELETE` 速度快，且使用的系统和事务日志资源少。但 `TRUNCATE`
   无事务且不触发 trigger，有可能造成事故，故不建议在开发代码中使用此语句。
   说明： `TRUNCATE TABLE` 在功能上与不带 `WHERE` 子句的 `DELETE` 语句相同。 




## 附录

### 字段名

用户名 user_name

创建时间 create_time

更新时间 modified_time



### 常量字段意义

1为android， 2为iOS，3为 web，4为H5