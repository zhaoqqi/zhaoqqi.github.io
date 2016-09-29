---
layout: post
title: Sql 的三种连接
categories: [Sql]
description: Sql 的三种连接，等值连接、左连接、右连接。
keywords: Sql, join, left join, right join
---

每次涉及到写复杂 sql 语句时，总是忘记等值连接、左连接、右连接这三种连接方式的区别。索性就写在这里，留着日后忘记了有个统一的地方查询。本文以 Mysql 数据库为例，进行简单描述和记录。

#### 左连接 - left join

先看个简单的 Sql 语句:
```sql
select a.id, a.name, b.address, b.age from a left join b on a.id = b.id and b.age > 18;

# 查询结果
1 Arya      Beijing 19
2 Snow      Beijing 20
3 ShangSha  NULL    NULL
```

左连接就是以"左表"的数据为主，如果在"左表"中的某一条记录，"右表"中不存在相应记录，则"右表"使用NULL填充。比如上面的 Sql 语句中的第三条记录，"左表"存在，"右表"不存在，则从"右表"中取的两个值由NULL填充。

#### 右连接 - right join

同样看个例子：
```sql
select b.id, a.name, b.address, b.age from a left join b on a.id = b.id and b.age > 18;

# 查询结果
1 NULL      Beijing  19 
2 NULL      Beijing  20
3 Halfman   Shanghai 32
```

左连接就是以"右表"的数据为主，如果在"左表"中的某一条记录，"左表"中不存在相应记录，则"左表"使用NULL填充。比如上面的 Sql 语句中的第三条记录，"右表"存在，"左表"不存在，则从"左表"中取的两个值由NULL填充。

#### 等值连接 - join

同样的 Sql 语句：
```sql
select b.id, a.name, b.address, b.age from a join b on a.id = b.id and b.age > 18;
或者
select b.id, a.name, b.address, b.age from a, b on a.id = b.id and b.age > 18;

# 查询结果
1 Arya      Beijing  19
2 Snow      Beijing  20
```

等值连接只会查询返回在两个表中均存在记录，其他的都会被过滤掉。

#### 总结

* 左连接就是以"左表"的查询记录条数为准，若对应"右表"的记录不存在，则以NULL值填充；
* 右连接就是以"右表"的查询记录条数为准，若对应"左表"的记录不存在，则以NULL值填充；
* 等值连接查询结果的条数，以"左表"和"右表"全都存在的记录为准，其他记录均过滤掉；

