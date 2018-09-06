# case语句

## case的用法

table t: 

index                               | age 
-                                   | - 
1                                   | 6
2                                   | 16
3                                   | 79

to

say                                 | count
-                                   | -
young                               | 2
old                                 | 1

```sql
select case when age > 60 then 'old' else 'young' end as say, count(1)
  from t
 group by case age when > 60 then 'old' else 'young' end
```
注意到求`say`的两行重复，在MySQL和postageSQL可以写：
```sql
select case when age > 60 then 'old' else 'young' end as say, count(1)
  from t
 group by say
```
这是不标准的SQL，这些数据库在执行查询语句时，会先对列进行计算。

## case表达式也能用在聚合函数中
```sql
select sum(case when age > 60 then age else 0 end) as sumOld
     , sum(case when age > 60 then 0 else age end) as sumYoung
  from t
```
## case表达式用在update的赋值中
```sql
update t
   set age = case when age > 60 then age else age + 1 end
```
也常用于交换值，比如将性别值交互
```sql
update t
   set sex = case sex when 'female' then 'male' else 'female' end
```

## 交叉显示

table t: 
index | age 
-     | - 
1     | 6
2     | 16
3     | 79

table r:
index | name 
-     | - 
1     | a
2     | b
3     | c

to

name | old | young 
-    | -   | -  
a    | n   | y
b    | n   | y
c    | y   | n

```sql
select name
     , case when index in (select index from t where age > 60) then 'y' else 'n' end as old
     , case when index in (select index from t where age > 60) then 'n' else 'y' end as young
  from r
```

# 自连接

## 全排列、组合
table t:

v | 
- | 
1 | 
2 | 
3 | 

to 

v1 | v2 | v3
-  | -  | -
1  | 2  | 3
1  | 3  | 2
2  | 1  | 3
2  | 3  | 1
3  | 1  | 2
3  | 2  | 1

```sql
select t1.v as v1, t2.v as v2, t3.v as v3
  from t t1, t t2, t t3
 where t1.v != t2.v and t1.v != t3.v and t2.v != t3.v
```

to

v1 | v2 | v3
-  | -  | -
1  | 2  | 3

```sql
select t1.v as v1, t2.v as v2, t3.v as v3
  from t t1, t t2, t t3
 where t1.v > t2.v and t2.v > t3.v
```


## 窗口函数

table t:
v |
- |
1 |
2 |
3 |
3 |
3 |
5 |

to

dense_rank | rank  | percent_rank | v
-          | -     | -            | -
1          | 1     | 0            | 1
2          | 2     | 0.2          | 2
3          | 3     | 0.4          | 3
3          | 3     | 0.4          | 3
3          | 3     | 0.4          | 3
**4**      | **6** | 1            | 5

```sql
select dense_rank() over(order by v) as dense_rank
     , rank() over(order by v) as rank
     , percent_rank() over(order by v) as percent_rank
     , v
  from t order by v;
```

# 三值逻辑（true/false/unknown）

结果总是`unknown`的表达式
```sql
null = null
col = null
col <> null
```

为什么对 NULL 使用比较谓词后得到的结果永远不可能为真呢？

    这是因为，NULL 既不是值也不是变量。NULL 只是一个表示“没有值”的标记，而比较谓词只适用于值。因此，对并非值的 NULL 使用比较谓词本来就是没有意义的
    “列的值为 NULL ”“NULL 值”这样的说法本身就是错误的。因为 NULL 不是值，所以不在定义域（domain）中。相反，如果有人认为 NULL 是值，那么笔者倒想请教一下：它是什么类型的值？关系数据库中存在的值必然属于某种类型，比如字符型或数值型等。所以，假如 NULL 是值，那么它就必须属于某种类型。

真值表
x | NOT x
- | -
t | f
u | u
f | t

AND | t | u | f
-   | - | - | -
t   | t | u | f
u   | u | u | f
f   | f | f | f

OR  | t | u | f
-   | - | - | -
t   | t | t | t
u   | t | u | u
f   | t | u | f

注意事项

    - 排中律不成立
    (col = 1 or col <> 1) => true or unknown
    
    - case表达式
    case col when null then 'null' else 'not null' end => 无论col是否null，表达式永远为'not null'
    
    - not in 和 not exists不等价
    not in aSet，取值 in aSet 取值 [unknown, true]，那么not in aSet 取值 [unknown, false]，而unknown取false的语义
    
    - 限定谓词all、any：
    ```select * from t where v < ALL (select v from t)``` 如果v取值里有null，就会出现unknown，那么结果是0行

    - 限定谓词与函数极值不等价
    ```select min(v) from t``` min函数将忽略null值 

    - 聚合函数和null
    ```sql
        SELECT *
        FROM Class_A
        WHERE age < ( SELECT AVG(age) FROM Class_B WHERE city = '东京' );
    ```
    当没有住在东京的记录，avg返回null

# having子句

## 是否有缺失的序列
```sql
select '存在缺失' from seqTable having count(*) <> max(seq) - min(seq) + 1
```

## 求众数

table t:
v |
- |
1 |
2 |
3 |
3 |
3 |
5 |

to

v | count(*)
- | -
3 | 3

```sql
select v, count(*) from t group by v having count(*) >= ALL(select count(*) from t group by v)
```

## 求中位数

table t:
v |
- |
1 |
2 |
3 |
3 |
3 |
5 |

to

v |
- |
3 |

```sql
select avg(distinct tmp.v)
  from (select t1.v
          from t t1, t t2
         group by t1.v
        having sum(case when t2.v >= t1.v then 1 else 0 end) >= count(*) / 2
           and sum(case when t2.v <= t1.v then 1 else 0 end) >= count(*) / 2) tmp
```

## 全部都有值的标签

table t:
tag | v
-   | -
a   | 1
a   | null
b   | 1
c   | 1
c   | 2

to

tag |
-   |
b   |
c   |

```sql
select tag from t group by t having count(*) = count(v)
```
```sql
select tag from t group by t having count(*) = sum(case when v is not null then 1 else 0 end)
```

## SQL除法

table t:
tag | v
-   | -
a   | 1
a   | 2
b   | 1
c   | 1
c   | 2
c   | 3

table r:
v |
- |
1 |
2 |

to

tag |
-   |
a   |
c   |

```sql
select t.tag
  from t, r
 where t.v = r.v
 group by t.tag
having count(t.v) = (select count(v) from r)
```

注意到c除了r表的所有值之外，还有一个值3, 如果结果不该出现c，称为 __精确关系除法__（exact relational division）

tag |
-   |
a   |

```sql
select t.tag
  from t left join r on t.v = r.v
 group by t.tag
having count(t.v) = (select count(v) from r)
   and count(r.v) = (select count(v) from r)
```

# 外连接

## 行 -> 列
table t:
tag | v
-   | -
a   | 1
a   | 3
b   | 2

to

tag | '1' | '2' | '3'
-   | -   | -   | -
a   | o   |     | o
b   |     | o   | 

```sql
select co.tag
     , case when co1.v is not null then 'o' else null end as '1'
     , case when co2.v is not null then 'o' else null end as '2'
     , case when co3.v is not null then 'o' else null end as '3'
  from (select distinct tag from t) co
  left join (select * from t where v = 1) as co1 on co.tag = co1.tag
  left join (select * from t where v = 2) as co2 on co.tag = co2.tag
  left join (select * from t where v = 3) as co3 on co.tag = co3.tag
```

通过子查询实现：

```sql
select co.tag
     , case when (select v from t where t.tag = co.tag and t.v = 1) is not null then 'o' else null end as '1'
     , case when (select v from t where t.tag = co.tag and t.v = 2) is not null then 'o' else null end as '2'
     , case when (select v from t where t.tag = co.tag and t.v = 3) is not null then 'o' else null end as '3'
  from (select distinct tag from t) co
```

## 列 -> 行

table t:
tag | v1 | v2
-   | -  | -
a   | 1  | 2
b   | 1  | 
c   |    | 

to

tag | v
-   | -
a   | 1
a   | 2
b   | 1
b   | null
c   | null
c   | null

```sql
select tag, v1 as v from t
union all
select tag, v2 as v from t
```

实际上，应该长成这样：

tag | v
-   | -
a   | 1
a   | 2
b   | 1
c   | null

引入table r:
v |
- |
1 |
2 |

```sql
select t.tag, r.v from t left join r on r.v in (t.v1, t.v2)
```
替换r表后：
```sql
select t.tag, r.v from t left join (select v1 as v from t union select v2 as v from t) r on r.v in (t.v1, t.v2)
```

## 作为乘法运算的连接

table t:
id | tag
-  | -
1  | a
2  | b

table r:
id | num
-  | -
1  | 2
1  | 3

求tag的数量和
to

id | num
-  | -
1  | 5
2  | 0

```sql
select t.id, rr.num
  from t
  left join (select id, sum(num) as num from r) rr
    on rr.id = t.id
```
性能有些问题，rr不存在主键索引，因此无法利用索引优化查询。
__优化：__
```sql
select t.id, sum(r.num) as num from t left join r on t.id = r.id
```

## 外连接、全外连接

MySQL不支持全外连接，用左右外连接和`union`可以达到类似的效果

```sql
select * from t left join r using(id)
union
select * from t right join r using(id)
```

引入集合的视角：

A - B
```sql
select * from A left join B using(id) where B.id is null
```

异或集：
`(A UNION B) EXCEPT (A INTERSECT B)`
或
`(A EXCEPT B) UNION (B EXCEPT A)`

```sql
select coalesce(A.id, B.id) from A full outer join B using(id) where A.id is null or B.id is null
```

# 关联子查询比较行与行

table t:
id | num
-  | -
1  | 3
2  | 4
3  | 4
4  | 2

to 

id | trending
-  | -
1  | -
2  | up
3  | stay
4  | down

使用关联子查询
```sql
select id
     , case when num = (select num from t t2 where t2.id = t1.id - 1) then 'stay'
            when num > (select num from t t2 where t2.id = t1.id - 1) then 'up'
            when num < (select num from t t2 where t2.id = t1.id - 1) then 'down'
            else '-' end as trending
  from t t1
```

改成列显示

to

id                                    | 1                             | 2                               | 3                               | 4
-                                     | -                             | -                               | -                               |  -
trending                              | -                             | up                              | stay                            | down
使用子查询
```sql
select 'trending' as id
     , (select case when num = (select num from t t2 where t2.id = t1.id - 1) then 'stay'
               when num > (select num from t t2 where t2.id = t1.id - 1) then 'up'
               when num < (select num from t t2 where t2.id = t1.id - 1) then 'down'
               else '-' end from t t1 where id = 1) as `1`
     , (select case when num = (select num from t t2 where t2.id = t1.id - 1) then 'stay'
               when num > (select num from t t2 where t2.id = t1.id - 1) then 'up'
               when num < (select num from t t2 where t2.id = t1.id - 1) then 'down'
               else '-' end from t t1 where id = 2) as `2`
     , (select case when num = (select num from t t2 where t2.id = t1.id - 1) then 'stay'
               when num > (select num from t t2 where t2.id = t1.id - 1) then 'up'
               when num < (select num from t t2 where t2.id = t1.id - 1) then 'down'
               else '-' end from t t1 where id = 3) as `3`
     , (select case when num = (select num from t t2 where t2.id = t1.id - 1) then 'stay'
               when num > (select num from t t2 where t2.id = t1.id - 1) then 'up'
               when num < (select num from t t2 where t2.id = t1.id - 1) then 'down'
               else '-' end from t t1 where id = 4) as `4`
```

如果id有间断：

table r:
id                              | num
-                               | -
1                               | 3
4                               | 4
5                               | 4
7                               | 2

to

pid                               | id                              | trending
-                                 | -                               | -
-                                 | 1                               | -
1                                 | 4                               | up
4                                 | 5                               | stay
5                                 | 7                               | down

```sql
select t2.id as pid, t1.id, t1.num
     , case when t1.num > t2.num then 'up'
       when t1.num < t2.num then 'down'
       when t1.num = t2.num then 'stay'
       else null end as trending
  from t t1
  left join t t2
    on t2.id = (select max(t3.id) from t t3 where t3.id < t1.id)
 order by t1.id;
```

## 移动累积值和平均值

table t:
id                              | num
-                               | -
1                               | 3
2                               | 4
3                               | 4
4                               | 2

to

id                              | num
-                               | -
1                               | 3
2                               | 7
3                               | 11
4                               | 13


使用窗口函数: 
```sql
select id, sum(num) over(order by num asc) as num from t
```
使用关联子查询
```sql
select id, (select sum(t1.num) from t t1 where t1.id <= t.id) as num from t
```

求相邻两个id的num和：

id                              | num
-                               | -
1                               | 3
2                               | 7
3                               | 8
4                               | 6

使用窗口函数：
```sql
select id, sum(num) over(order by id rows 1 preceding) as num from t
```
使用关联子查询
```sql
select id
     , (select sum(t1.num)
          from t t1
         where t1.id <= t.id
           and (select count(1)
          from t t2
         where t2.id between t1.id and t.id) <= 2) as num
  from t
```

## 查询重叠的区间

table t:
id                              | start                             | end
-                               | -                                 | -
1                               | 3                                 | 6
2                               | 4                                 | 5
3                               | 7                                 | 9
4                               | 8                                 | 9
5                               | 1                                 | 2

to

id                              |
-                               |
1                               |
2                               |
3                               |
4                               |

```sql
select id
  from t
 where exists (select *
                 from t t1
                where t.id <> t1.id
                  and (t.start between t1.start and t1.end
                      or t.end between t1.start and t1.end))
```
输出
id                              |
-                               |
2                               |
3                               |
4                               |
改进：

```sql
select id
  from t
 where exists (select *
                 from t t1
                where t.id <> t1.id
                  and ((t.start between t1.start and t1.end or t.end between t1.start and t1.end)
                      or (t1.start between t.start and t.end and t1.end between t.start and t.end)))
```

## 用SQL进行集合运算

除法运算的实现

- 嵌套使用NOT EXISTS
- 使用HAVING子句转换成一对一关系
- 把除法变成减法
  ```sql
  -- data(id, name) / ids = name
  select distinct name from data d1 where not exists (select id from ids except select id from data d2 where d1.name = d2.name)
  ```

## 寻找相等的子集

table t:
tag                               | id
-                                 | -
a                                 | 1
a                                 | 2
b                                 | 1
b                                 | 2
c                                 | 1
d                                 | 1
d                                 | 2
d                                 | 3

to

tag1                              | tag2
-                                 | -
a                                 | b

```sql
select t1.tag, t2.tag
  from t t1, t t2
 where t1.tag < t2.tag and t1.id = t2.id
 group by t1.tag, t2.tag
having count(*) = (select count(*) from t t3 where t3.tag = t1.tag)
   and count(*) = (select count(*) from t t4 where t4.tag = t2.tag)
```

# EXISTS谓词的用法

> 谓词是一种特殊的函数，返回值是真值。前面提到的每个谓词，返回值都是 true 、 false 或者 unknown （一般的谓词逻辑里没有 unknown ，但是 SQL 采用的是三值逻辑，因此具有三种真值）。
> 
> 谓词逻辑提供谓词是为了判断命题（可以理解成陈述句）的真假。
> 
> EXISTS 可以看成是一种高阶函数

table t:
no                              | tag
-                               | -
1                               | a
1                               | b
1                               | c
2                               | c
2                               | d

to

no                              | tag
-                               | -
1                               | d
2                               | a
2                               | b

```sql
select distinct t1.no, t2.tag
  from t t1, t t2
 where not exists (select * from t t3 where t3.no = t1.no and t3.tag = t2.tag)
```
或者减法:
```sql
select distinct t1.no, t2.tag
  from t t1, t t2
except select no, tag from t
```

全称量化
所有行满足 => 不满足的行都不存在

# 用SQL处理数列

## 生成连续编号

table d:
s                               |
-                               |
0                               |
1                               |
2                               |
3                               |
4                               |
5                               |
6                               |
7                               |
8                               |
9                               |

to

s                                 |
-                                 |
0                                 |
1                                 |
2                                 |
...                               |
98                                |
99                                |

```sql
select (d1.s + s2.s*10) as s from d d1, d d2 order by s;
```

类似的，可以生成0~999的序列

## 是否连续三个空位
table t:
id                                | lock
1                                 | 1
2                                 | 1
3                                 | 0
4                                 | 0
5                                 | 0
6                                 | 1
7                                 | 0
8                                 | 0
9                                 | 0
10                                | 0
11                                | 0
12                                | 1
13                                | 1
14                                | 0
15                                | 0

to

start                               | end
-                                   | -
3                                   | 5
7                                   | 9
8                                   | 10
9                                   | 11

思路：在这些区间里没有lock=1的记录
```sql
select t1.id as `start`, t2.id as `end`
  from t t1, t t2
 where t2.id = t1.id + (3 - 1)
   and not exists (select 1 from t t3 where t3.id between t1.id and t2.id and t3.lock <> 0)
```

## 连续空位个数
to
start                               | end                               | cnt
-                                   |   -                               | -
3                                   | 5                                 | 3
7                                   | 11                                | 5
14                                  | 15                                | 2

```sql
select t1.id as `start`, t2.id as `end`, t2.id - t1.id + 1 as `cnt`
  from t t1, t t2
 where t1.id < t2.id
   and not exists (select 1 from t t3
                    where (t3.id between t1.id and t2.id and t3.lock <> 0)
                       or (t3.id = t2.id + 1 and t3.lock = 0)
                       or (t3.id = t1.id - 1 and t3.lock = 0))
```

# HAVING子句又回来了

> 强调 SQL 的处理单位不是记录，而是集合。 ——Joe Celko

条件表达式                              | 用途
-                                      | -
COUNT (DISTINCT col) = COUNT (col)     |  col 列没有重复的值
COUNT(*) = COUNT(col)                  | col 列不存在 NULL
COUNT(*) = MAX(col) - MIN(col) + 1     | col 列是连续的编号（起始值是任意整数）
MIN(col) = MAX(col)                    | col 列都是相同值，或者是 NULL
MIN(col) * MAX(col) > 0                | col 列全是正数或全是负数
MIN(col) * MAX(col) < 0                | col 列的最大值是正数，最小值是负数
MIN(ABS(col)) = 0                      | col 列最少有一个是 0
MIN(col - 常量 ) = - MAX(col - 常量 )   | col 列的最大值和最小值与指定常量等距 

# 让SQL飞起来

## 使用高效的查询
- 参数是子查询，使用EXISTS替代IN
- 参数是子查询，使用连接代替 IN 

## 避免排序

常见的排序运算
- group by子句
- order by子句
- 聚合函数（sum、count、avg、max、min）
- distinct
- 集合运算符（union、intersect、except）
- 窗口函数（rank、row_number等）

建议
- 灵活使用集合运算符的ALL可选项
- 使用exists替代distinct


table t:

id |
-  |
1  |
2  |
3  |

table r:
id |
-  |
1  |
1  |
2  |

to

id |
-  |
1  |
2  |

```sql
select distinct t.id from t join r using(id)
```
使用exists替代
```sql
selec t.id from t where exists (select 1 from t t1 where t1.id = t.id)
```
- 在极值函数种使用索引（`select min(index_col)`）
- 能写在where子句的条件不要写在having子句里
> HAVING 子句是针对聚合后生成的视图进行筛选的，但是很多时候聚合后的视图都没有继承原表的索引结构 。

- 在 GROUP BY 子句和 ORDER BY 子句中使用索引

## 真的用到索引了吗
- 索引字段上进行运算

> 使用索引时，条件表达式的左侧应该是原始字段 

```sql
WHERE col_1 * 1.1 > 100 -- x
WHERE col_1 > 100 / 1.1 -- ✔
```

```sql
WHERE SUBSTR(col_1, 1, 1) = 'a'; -- x
WHERE col_1 like 'a%'            -- ✔
```
- 使用 IS NULL 谓词

```sql
WHERE col_1 IS NULL;  -- DB2 和 Oracle 支持null的索引，但并非所有数据库都支持
```

- 使用否定形式

  - <>
  - !=
  - not in

- 使用 OR
- 使用联合索引时，列的顺序错误
- 使用 LIKE 谓词进行后方一致或中间一致的匹配
- 进行默认的类型转换


## 减少中间表

- 灵活使用 HAVING 子句

常见的问题是通过group by生成中间表，然后进行where条件筛选，其实可以使用having子句

- 需要对多个字段使用 IN 谓词时，将它们汇总到一处

```sql
select ('a', 1) in (select tag, id from t)
select ('a', 1) in (select tag, id from t)
```

- 先进行连接再进行聚合

- 合理地使用视图
> 特别是视图的定义语句中包含以下运算的时候，SQL 会非常低效，执行速度也会变得非常慢。
> 
> - 聚合函数（AVG 、COUNT 、SUM 、MIN 、MAX ） 
> - 集合运算符（UNION 、*NTERSECT 、EXCEPT 等） 
> 
> 一般来说，要格外注意避免在视图中进行聚合操作后需要特别注意 。

# SQL编程方法

## 表的设计

- 名字和意义 
- 属性和列

## 编程的方针
- 注释
```sql
-- 单行
/*
多行
*/
```

- 缩进
- 空格
- 大小写 
- 逗号
- 不使用通配符

```sql
SELECT * FROM SomeTable;                     -- x
SELECT col_1, col2, col3 ... FROM SomeTable; -- ✔
```
- ORDER BY 中不使用列编号
```sql
SELECT col_1, col2 FROM SomeTable ORDER BY 1, 2;       -- ×
SELECT col_1, col2 FROM SomeTable ORDER BY col_1, col2 -- √
```
## SQL编程方法

- 使用SQL标准

  1. 不使用依赖各种数据库实现的函数和运算符 
  2. 连接操作使用标准语法 
- “左派”和“右派” （左外连接/右外连接）
- 从 FROM 子句开始写 























