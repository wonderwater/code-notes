# Mysql数据库事务实验

实验条件：  
Mysql 5.7.21-0ubuntu0.16.04.1-log  
可重复读  
准备数据：

```sql
select * from stock;
```

+----+--------+-----+  
\| id \| sku\_id \| num \|  
+----+--------+-----+  
\|  1 \|      1 \|  10 \|  
+----+--------+-----+

* 第一个例子：当前事务中的update语句好像不生效

事务1：

```sql
begin;
select num from stock;    -- 10
update stock set num = num - 7 where num >= 7; -- Rows matched: 1  Changed: 1  Warnings: 0
select num from stock;    -- 3
```

事务2：

```sql
begin;
select num from stock;    -- 10
update stock set num = num - 7 where num >= 7; -- blocking
```

事务1提交后：

```sql
commit;
select num from stock;    -- 3
```

事务2：num结果是什么？

```sql
select num from stock;    -- 10
update stock set num = num - 7 where num >= 7; -- ????
```

事务2结果：

```sql
update stock set num = num - 7 where num >= 7; -- Rows matched: 0  Changed: 0  Warnings: 0
select num from stock;    -- 10
```

结果是10，在事务2里看起来好像update语句不生效。

事务2提交后：

```sql
commit;
select num from stock;    -- 3
```

* 第二个例子：当前事务中的update语句好像修改错了

将数据还原:

```sql
update stock set num = 10;
```

事务3：

```sql
begin;
select num from stock;    -- 10
update stock set num = num - 3 where num >= 3; -- Rows matched: 1  Changed: 1  Warnings: 0
select num from stock;    -- 7
```

事务4：

```sql
begin;
select num from stock;    -- 10
update stock set num = num - 3 where num >= 3; -- blocking
```

事务3提交后：

```sql
commit;
select num from stock;    -- 7
```

事务4：num结果是什么？

```sql
select num from stock;    -- 10
update stock set num = num - 3 where num >= 3; -- ????
```

事务4结果：

```sql
update stock set num = num - 7 where num >= 7; -- Rows matched: 1  Changed: 1  Warnings: 0
select num from stock;    -- 4
```

结果是4

事务4提交后：

```sql
commit;
select num from stock;    -- 4
```

> 这说明了：在一个事务内始终保持可重复读，但是更新操作始终以提交版本为准。

下面列举几项相关现象，问题命名是根据t1事务的现象而来的：

1.mysql事务在事务里更新不存在的记录

|t1|t2|
|:--|:--|
|begin;|begin;|
|select * from stock; -- empty||
||insert into stock (id, sku_id, num) values (2,2,2); -- Rows matched: 1  Changed: 1|
||select num from stock; -- 2,2,2|
|select * from stock; -- empty||
|update stock set num = 1 where id = 2; -- blocking||
||commit;|
|update stock set num = 1 where id = 2; -- Rows matched: 1  Changed: 1||
|select * from stock; -- 2,2,1||
||select num from stock; -- 2,2,2|
|commit;||
|select * from stock; -- 2,2,1||

2.mysql事务在事务里并不能保证一致性

|t1|t2|
|:--|:--|
|begin;|begin;|
|select * from stock; -- 1,1,10\|2,2,1|select * from stock; -- 1,1,10\|2,2,1|
||update stock set num = num + 1; -- Rows matched: 2  Changed: 2|
||select * from stock; -- 1,1,11\|2,2,2|
|update stock set num = num + 1 where id = 1; -- blocking||
||commit;|
|update stock set num = num + 1 where id = 1; -- Rows matched: 1  Changed: 1||
||select * from stock; -- 1,1,11\|2,2,2|
|select * from stock; -- 1,1,12\|2,2,1||
|update stock set num = num + 0; -- Rows matched: 2  Changed: 0||
|select * from stock; -- 1,1,12\|2,2,1||
|commit;||
|select * from stock; -- 1,1,12\|2,2,2||

3.mysql事务在事务里好像update没生效

|t1|t2|
|:--|:--|
|begin;|begin;|
|select num from stock; -- 10|select num from stock; -- 10|
||update stock set num = num - 7 where num >= 7; -- Rows matched: 1 Changed: 1 Warnings: 0|
|update stock set num = num - 7 where num >= 7; -- blocking||
||select num from stock; -- 3|
||commit;|
|update stock set num = num - 7 where num >= 7; -- Rows matched: 0 Changed: 0 Warnings: 0||
||select num from stock; -- 3|
|select num from stock; -- 10||
|commit;||
|select num from stock; -- 3||

4.mysql事务在事务里好像update执行错了

|t1|t2|
|:--|:--|
|begin;|begin;|
|select num from stock; -- 10|select num from stock; -- 10|
||update stock set num = num - 3 where num >= 3; -- Rows matched: 1 Changed: 1 Warnings: 0|
||select num from stock; -- 7|
|update stock set num = num - 3 where num >= 3; -- blocking||
||commit;|
|update stock set num = num - 7 where num >= 7; -- Rows matched: 1 Changed: 1 Warnings: 0||
||select num from stock; -- 7|
|select num from stock; -- 4||
|commit;||
|select num from stock;    -- 4||

