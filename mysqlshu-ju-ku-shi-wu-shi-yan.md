# Mysql数据库事务实验

实验条件：  
Mysql 5.7.21-0ubuntu0.16.04.1-log  
可重复读  
准备数据：

```
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

```
begin;
select num from stock;    -- 10
update stock set num = num - 7 where num >= 7; -- blocking
```

事务1提交后：

```
commmit;
select num from stock;    -- 3
```

事务2：num结果是什么？

```
select num from stock;    -- 10
update stock set num = num - 7 where num >= 7; -- ????
```

事务2结果：

```
update stock set num = num - 7 where num >= 7; -- Rows matched: 0  Changed: 0  Warnings: 0
select num from stock;    -- 10
```

结果是10，在事务2里看起来好像update语句不生效。

事务2提交后：

```
commmit;
select num from stock;    -- 3
```

* 第二个例子：当前事务中的update语句好像修改错了

将数据还原: 

```
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

```
begin;
select num from stock;    -- 10
update stock set num = num - 3 where num >= 3; -- blocking
```

事务3提交后：

```
commmit;
select num from stock;    -- 7
```

事务4：num结果是什么？

```
select num from stock;    -- 10
update stock set num = num - 3 where num >= 3; -- ????
```

事务4结果：

```
update stock set num = num - 7 where num >= 7; -- Rows matched: 1  Changed: 1  Warnings: 0
select num from stock;    -- 4
```

结果是4

事务4提交后：

```
commmit;
select num from stock;    -- 4
```

这说明了：在一个事务内始终保持可重复读，但是更新操作始终以提交版本为准。







