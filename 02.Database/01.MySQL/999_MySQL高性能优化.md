# MySQL高性能优化

## 数据库命令规范

1. 所有数据库对象名称必须使用小写字母并用下划线分割

2. 所有数据库对象名称禁止使用 MySQL 保留关键字（如果表名中包含关键字查询时，需要将其用单引号括起来）

3. 数据库对象的命名要能做到见名识意，并且最后不要超过 32 个字符

4. 临时库表必须以 tmp_为前缀并以日期为后缀，备份表必须以 bak_为前缀并以日期（时间戳）为后缀

5. 所有存储相同数据的列名和列类型必须一致（一般作为关联列，如果查询时关联列类型不一致会自动进行数据类型隐式转换，会造成列上的索引失效，导致查询效率降低）

## 数据库基本设计规范

1. 所有表必须使用 InnoDB 存储引擎

   InnoDB 支持事务，支持行级锁，更好的恢复性，高并发下性能更好

2. 数据库和表的字符集统一使用 UTF8

   兼容性更好，统一字符集可以避免由于字符集转换产生的乱码，不同的字符集进行比较前需要进行转换会造成索引失效

3. 所有表和字段都需要添加注释

   使用 comment 从句添加表和列的备注，从一开始就进行数据字典的维护

4. 尽量控制单表数据量的大小，建议控制在 500 万以内

   可以用历史数据归档（应用于日志数据），分库分表（应用于业务数据）等手段来控制数据量大小

5. 谨慎使用 MySQL 分区表

6. 尽量做到冷热数据分离,减小表的宽度

   >MySQL 限制每个表最多存储 4096 列，并且每一行数据的大小不能超过 65535 字节。

7. 禁止在表中建立预留字段

8. 禁止在数据库中存储图片,文件等大的二进制数据

9. 禁止在线上做数据库压力测试

10. 禁止从开发环境,测试环境直接连接生成环境数据库

## 数据库字段设计规范

1. 优先选择符合存储需要的最小的数据类型

   列的字段越大，建立索引时所需要的空间也就越大，这样一页中所能存储的索引节点的数量也就越少也越少，在遍历时所需要的 IO 次数也就越多，索引的性能也就越差。

2. 避免使用 TEXT,BLOB 数据类型，最常见的 TEXT 类型可以存储 64k 的数据

3. 避免使用 ENUM 类型

4. 尽可能把所有列定义为 NOT NULL

   * 索引 NULL 列需要额外的空间来保存，所以要占用更多的空间

   * 进行比较和计算时要对 NULL 值做特别的处理

5. 使用 TIMESTAMP(4 个字节) 或 DATETIME 类型 (8 个字节) 存储时间

6. 同财务相关的金额类数据必须使用 decimal 类型

## 索引设计规范

1. 限制每张表上的索引数量,建议单张表索引不超过 5 个

2. 禁止给表中的每一列都建立单独的索引

3. 每个 InnoDB 表必须有个主键

4. 常见索引列建议
   * 出现在 SELECT、UPDATE、DELETE 语句的 WHERE 从句中的列
   * 包含在 ORDER BY、GROUP BY、DISTINCT 中的字段
   * 并不要将符合 1 和 2 中的字段的列都建立一个索引， 通常将 1、2 中的字段建立联合索引效果更好
   * 多表 join 的关联列

5. 如何选择索引列的顺序

   建立索引的目的是：希望通过索引进行数据查找，减少随机 IO，增加查询性能 ，索引能过滤出越少的数据，则从磁盘中读入的数据也就越少。

   * 区分度最高的放在联合索引的最左侧（区分度=列中不同值的数量/列的总行数）
   * 尽量把字段长度小的列放在联合索引的最左侧（因为字段长度越小，一页能存储的数据量越大，IO 性能也就越好）
   * 使用最频繁的列放到联合索引的左侧（这样可以比较少的建立一些索引）

6. 避免建立冗余索引和重复索引（增加了查询优化器生成执行计划的时间）

   * 重复索引示例：primary key(id)、index(id)、unique index(id)
   * 冗余索引示例：index(a,b,c)、index(a,b)、index(a)

7. 对于频繁的查询优先考虑使用覆盖索引

8. 索引 SET 规范

   尽量避免使用外键约束

   * 不建议使用外键约束（foreign key），但一定要在表与表之间的关联键上建立索引
   * 外键可用于保证数据的参照完整性，但建议在业务端实现
   * 外键会影响父表和子表的写操作从而降低性能

## 数据库 SQL 开发规范

1. 建议使用预编译语句进行数据库操作

2. 避免数据类型的隐式转换

   隐式转换会导致索引失效

3. 充分利用表上已经存在的索引

4. 数据库设计时，应该要对以后扩展进行考虑

5. 程序连接不同的数据库使用不同的账号，进制跨库查询

6. 禁止使用 SELECT * 必须使用 SELECT <字段列表> 查询

   * 消耗更多的 CPU 和 IO 以网络带宽资源
   * 无法使用覆盖索引
   * 可减少表结构变更带来的影响

7. 禁止使用不含字段列表的 INSERT 语句

8. 避免使用子查询，可以把子查询优化为 JOIN 操作

9. 避免使用 JOIN 关联太多的表

10. 减少同数据库的交互次数

11. 对应同一列进行 OR 判断时，使用 IN 代替 OR

    IN 的值不要超过 500 个，IN 操作可以更有效的利用索引，OR 大多数情况下很少能利用到索引。

12. 禁止使用 order by rand() 进行随机排序

    推荐在程序中获取一个随机值，然后从数据库中获取数据的方式。

13. WHERE 从句中禁止对列进行函数转换和计算

    对列进行函数转换或计算时会导致无法使用索引

14. 在明显不会有重复值时使用 UNION ALL 而不是 UNION

    * UNION ALL 不会再对结果集进行去重操作
    * UNION 会把两个结果集的所有数据放到临时表中后再进行去重操作

15. 拆分复杂的大 SQL 为多个小 SQL

## 数据库操作行为规范

1. 超 100 万行的批量写（INSERT，DELETE，UPDATE）操作，要分批多次进行操作

   * 大批量操作可能会造成严重的主从延迟
   * binlog 日志为 row 格式时会产生大量的日志
   * 避免产生大事务操作

2. 对于大表使用 pt-online-schema-change 修改表结构

   * 避免大表修改产生的主从延迟
   * 避免在对表字段进行修改时进行锁表

3. 禁止为程序使用的账号赋予 super 权限

   * 当达到最大连接数限制时，还运行 1 个有 super 权限的用户连接
   * super 权限只能留给 DBA 处理问题的账号使用

4. 对于程序连接数据库账号，遵循权限最小原则

   * 程序使用数据库账号只能在一个 DB 下使用，不准跨库
   * 程序使用的账号原则上不准有 drop 权限
