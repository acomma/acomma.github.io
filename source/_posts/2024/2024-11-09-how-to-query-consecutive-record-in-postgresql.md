---
title: 在 PostgreSQL 如何查询连续出现的记录
date: 2024-11-09 22:11:37
updated: 2024-11-09 22:11:37
tags: [PostgreSQL]
---

## 场景描述

我们有一个系统在不停接收其他系统发来的消息，我们的任务任务是查询连续出现两次及以上的消息。下面是一些例子

1. 假设依次收到的消息是 A 则结果为空；
2. 假设依次收到的消息是 AB 则结果为空；
3. 假设依次收到的消息是 AA 则结果为 AA；
4. 假设依次收到的消息是 AABB 则结果为 AABB；
5. 假设依次收到的消息是 ABB 则结果为 BB；
6. 假设依次收到的消息是 AAB 则结果为 AA；
7. 假设依次收到的消息是 AABCC 则结果为 AACC；
8. 假设依次收到的消息是 AABCDD 则结果为 AADD；
9. 假设依次收到的消息是 AABAACC 则结果为 AAAACC；
10. 假设依次收到的消息是 AABAABCC 则结果为 AAAACC；
11. 假设依次收到的消息是 AABAACDD 则结果为 AAAADD；
12. 假设依次收到的消息是 AABCCBDD 则结果为 AACCDD；

<!-- more -->

## 数据准备

下面我们创建一张消息表 `message` 来模拟这种情况

```sql
CREATE TABLE message (
    id SERIAL4,
    sender VARCHAR,
    content VARCHAR,
    create_at TIMESTAMP(0),
    PRIMARY KEY (id)
);

INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:11');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:12');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'B', '2024-11-09 11:11:13');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'B', '2024-11-09 11:11:14');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'B', '2024-11-09 11:11:15');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'C', '2024-11-09 11:11:16');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:17');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:18');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:19');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'D', '2024-11-09 11:11:20');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'D', '2024-11-09 11:11:21');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'D', '2024-11-09 11:11:22');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'E', '2024-11-09 11:11:23');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:24');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:25');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'B', '2024-11-09 11:11:26');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:27');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:28');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'C', '2024-11-09 11:11:29');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'C', '2024-11-09 11:11:30');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'D', '2024-11-09 11:11:31');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'D', '2024-11-09 11:11:32');
```

在这份样例数据里，没有连续出现两次及以上的记录是第 14 行 Bob 发送的消息 C，第 21 行 Bob 发送的消息 E 和第 24 行 Alice 发送的消息 B。

## 方法一

对于任意一条记录来说，它的内容连续出现两次及以上的条件是它的内容要么和前一条记录的内容一样，要么和后一条记录的内容一样。在 PostgreSQL 中可以通过 `LAG` 函数获取前一条记录的内容，通过 `LEAD` 函数获取后一条记录的内容

```sql
SELECT *,
       LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
       LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
FROM message
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+------------+
|id|sender|content|create_at          |prev_content|next_content|
+--+------+-------+-------------------+------------+------------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |A           |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |B           |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |B           |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |B           |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |C           |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |A           |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |A           |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |A           |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |D           |
|10|Bob   |D      |2024-11-09 11:11:20|A           |D           |
|11|Bob   |D      |2024-11-09 11:11:21|D           |D           |
|12|Bob   |D      |2024-11-09 11:11:22|D           |E           |
|13|Bob   |E      |2024-11-09 11:11:23|D           |null        |
|14|Alice |A      |2024-11-09 11:11:24|null        |A           |
|15|Alice |A      |2024-11-09 11:11:25|A           |B           |
|16|Alice |B      |2024-11-09 11:11:26|A           |A           |
|17|Alice |A      |2024-11-09 11:11:27|B           |A           |
|18|Alice |A      |2024-11-09 11:11:28|A           |C           |
|19|Alice |C      |2024-11-09 11:11:29|A           |C           |
|20|Alice |C      |2024-11-09 11:11:30|C           |D           |
|21|Alice |D      |2024-11-09 11:11:31|C           |D           |
|22|Alice |D      |2024-11-09 11:11:32|D           |null        |
+--+------+-------+-------------------+------------+------------+
```

我们只需要找到 `content` 要么等于 `prev_content` 要么等于 `next_content` 的记录，这些记录就是连续出现两次及以上的记录

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *,
           LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
           LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
    FROM message
) AS t
WHERE content = prev_content OR content = next_content
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+
|id|sender|content|create_at          |
+--+------+-------+-------------------+
|1 |Bob   |A      |2024-11-09 11:11:11|
|2 |Bob   |A      |2024-11-09 11:11:12|
|3 |Bob   |B      |2024-11-09 11:11:13|
|4 |Bob   |B      |2024-11-09 11:11:14|
|5 |Bob   |B      |2024-11-09 11:11:15|
|7 |Bob   |A      |2024-11-09 11:11:17|
|8 |Bob   |A      |2024-11-09 11:11:18|
|9 |Bob   |A      |2024-11-09 11:11:19|
|10|Bob   |D      |2024-11-09 11:11:20|
|11|Bob   |D      |2024-11-09 11:11:21|
|12|Bob   |D      |2024-11-09 11:11:22|
|14|Alice |A      |2024-11-09 11:11:24|
|15|Alice |A      |2024-11-09 11:11:25|
|17|Alice |A      |2024-11-09 11:11:27|
|18|Alice |A      |2024-11-09 11:11:28|
|19|Alice |C      |2024-11-09 11:11:29|
|20|Alice |C      |2024-11-09 11:11:30|
|21|Alice |D      |2024-11-09 11:11:31|
|22|Alice |D      |2024-11-09 11:11:32|
+--+------+-------+-------------------+
```

### 连续出现三次及以上的记录

对于任意一条记录来说，它的内容连续出现三次及以上的条件是它的内容要么和前两条记录的内容一样，要么和后两条记录的内容一样，要么和前一条和后一条的内容都一样。`LAG` 函数的完整定义为 `lag(value anyelement [, offset integer [, default anyelement ]])`，它在分区内当前行的之前 `offset` 个位置的行上计算，`offset` 的默认值为 1。可以将 `offset` 的值设为 2 从而得到在当前记录前面第二条的记录。`LEAD` 函数是类似的。

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *,
           LAG(content, 1) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content_1,  -- 当前行前面第一行的内容
           LAG(content, 2) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content_2,  -- 当前行前面第二行的内容
           LEAD(content, 1) OVER(PARTITION BY sender ORDER BY create_at) AS next_content_1, -- 当前行后面第一行的内容
           LEAD(content, 2) OVER(PARTITION BY sender ORDER BY create_at) AS next_content_2  -- 当前行后面第二行的内容
    FROM message
) AS t
WHERE (content = prev_content_1 AND content = prev_content_2) -- 当前行内容与它前面两行内容一样
   OR (content = next_content_1 AND content = next_content_2) -- 当前行内容与它后面两行内容一样
   OR (content = prev_content_1 AND content = next_content_1) -- 当前行内容与它前面一行和后面一行内容一样
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+
|id|sender|content|create_at          |
+--+------+-------+-------------------+
|3 |Bob   |B      |2024-11-09 11:11:13|
|4 |Bob   |B      |2024-11-09 11:11:14|
|5 |Bob   |B      |2024-11-09 11:11:15|
|7 |Bob   |A      |2024-11-09 11:11:17|
|8 |Bob   |A      |2024-11-09 11:11:18|
|9 |Bob   |A      |2024-11-09 11:11:19|
|10|Bob   |D      |2024-11-09 11:11:20|
|11|Bob   |D      |2024-11-09 11:11:21|
|12|Bob   |D      |2024-11-09 11:11:22|
+--+------+-------+-------------------+
```

## 方法二

方法一如果要查询连续出现更多次的记录则需要更多的 `LAG` 和 `LEAD` 函数，如果把连续出现的次数作为一个参数则需要动态的拼接查询语句。下面来实现一种不需要动态拼接查询语句即可实现连续出现任意次数的需求。

观察前面的样例数据，一种直观想法是，为了查询连续出现的记录，只需要根据消息内容分组，统计每个分组的数量，筛选出数量大于等于 2 的那些分组即是我们需要的结果。但是这种实现方式无法处理 AABCCBDD 这种情形，按照这种实现方式 B 将被保留，结果为 AABCCBDD，而我们期望的结果为 AACCDD。因此需要寻找一种根据其他虚拟（或计算）字段进行分组的方法。

为了比较前后两条消息是否有变化，需要使用 `LAG` 函数获取当前消息的前一条消息

```sql
SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
FROM message
ORDER BY id;
```

接下来是判断当前消息相对于前一条消息是否有变化，需要用到 `CASE ... WHEN ... THEN ... ELSE ... END` 表达式，0 表示没有变化，1 表示有变化

```sql
SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
FROM (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM message
) AS preved
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+---------+
|id|sender|content|create_at          |prev_content|is_change|
+--+------+-------+-------------------+------------+---------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |1        |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |0        |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |1        |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |0        |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |0        |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |1        |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |1        |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |0        |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |0        |
|10|Bob   |D      |2024-11-09 11:11:20|A           |1        |
|11|Bob   |D      |2024-11-09 11:11:21|D           |0        |
|12|Bob   |D      |2024-11-09 11:11:22|D           |0        |
|13|Bob   |E      |2024-11-09 11:11:23|D           |1        |
|14|Alice |A      |2024-11-09 11:11:24|null        |1        |
|15|Alice |A      |2024-11-09 11:11:25|A           |0        |
|16|Alice |B      |2024-11-09 11:11:26|A           |1        |
|17|Alice |A      |2024-11-09 11:11:27|B           |1        |
|18|Alice |A      |2024-11-09 11:11:28|A           |0        |
|19|Alice |C      |2024-11-09 11:11:29|A           |1        |
|20|Alice |C      |2024-11-09 11:11:30|C           |0        |
|21|Alice |D      |2024-11-09 11:11:31|C           |1        |
|22|Alice |D      |2024-11-09 11:11:32|D           |0        |
+--+------+-------+-------------------+------------+---------+
```

现在使用 `SUM` 窗口函数对 `is_change` 列求和，求和的结果作为分组 ID

```sql
SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
FROM (
    SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
    FROM (
        SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
        FROM message
    ) AS preved
) AS content_change
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+---------+--------+
|id|sender|content|create_at          |prev_content|is_change|group_id|
+--+------+-------+-------------------+------------+---------+--------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |1        |1       |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |0        |1       |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |1        |2       |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |0        |2       |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |0        |2       |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |1        |3       |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |1        |4       |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |0        |4       |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |0        |4       |
|10|Bob   |D      |2024-11-09 11:11:20|A           |1        |5       |
|11|Bob   |D      |2024-11-09 11:11:21|D           |0        |5       |
|12|Bob   |D      |2024-11-09 11:11:22|D           |0        |5       |
|13|Bob   |E      |2024-11-09 11:11:23|D           |1        |6       |
|14|Alice |A      |2024-11-09 11:11:24|null        |1        |1       |
|15|Alice |A      |2024-11-09 11:11:25|A           |0        |1       |
|16|Alice |B      |2024-11-09 11:11:26|A           |1        |2       |
|17|Alice |A      |2024-11-09 11:11:27|B           |1        |3       |
|18|Alice |A      |2024-11-09 11:11:28|A           |0        |3       |
|19|Alice |C      |2024-11-09 11:11:29|A           |1        |4       |
|20|Alice |C      |2024-11-09 11:11:30|C           |0        |4       |
|21|Alice |D      |2024-11-09 11:11:31|C           |1        |5       |
|22|Alice |D      |2024-11-09 11:11:32|D           |0        |5       |
+--+------+-------+-------------------+------------+---------+--------+
```

接下来使用新的分组 ID 统计每个分组的数量

```sql
SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
FROM (
    SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
    FROM (
        SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
        FROM (
            SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
            FROM message
        ) AS preved
    ) AS content_change
) AS grouped
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+---------+--------+-----------+
|id|sender|content|create_at          |prev_content|is_change|group_id|group_count|
+--+------+-------+-------------------+------------+---------+--------+-----------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |1        |1       |2          |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |0        |1       |2          |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |1        |2       |3          |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |0        |2       |3          |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |0        |2       |3          |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |1        |3       |1          |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |1        |4       |3          |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |0        |4       |3          |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |0        |4       |3          |
|10|Bob   |D      |2024-11-09 11:11:20|A           |1        |5       |3          |
|11|Bob   |D      |2024-11-09 11:11:21|D           |0        |5       |3          |
|12|Bob   |D      |2024-11-09 11:11:22|D           |0        |5       |3          |
|13|Bob   |E      |2024-11-09 11:11:23|D           |1        |6       |1          |
|14|Alice |A      |2024-11-09 11:11:24|null        |1        |1       |2          |
|15|Alice |A      |2024-11-09 11:11:25|A           |0        |1       |2          |
|16|Alice |B      |2024-11-09 11:11:26|A           |1        |2       |1          |
|17|Alice |A      |2024-11-09 11:11:27|B           |1        |3       |2          |
|18|Alice |A      |2024-11-09 11:11:28|A           |0        |3       |2          |
|19|Alice |C      |2024-11-09 11:11:29|A           |1        |4       |2          |
|20|Alice |C      |2024-11-09 11:11:30|C           |0        |4       |2          |
|21|Alice |D      |2024-11-09 11:11:31|C           |1        |5       |2          |
|22|Alice |D      |2024-11-09 11:11:32|D           |0        |5       |2          |
+--+------+-------+-------------------+------------+---------+--------+-----------+
```

在上面结果的基础上排除分组数量为 1 的记录

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
    FROM (
        SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
        FROM (
            SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
            FROM (
                SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
                FROM message
            ) AS preved
        ) AS content_change
    ) AS grouped
) AS counted
WHERE group_count > 1
ORDER BY id;
```

这个查询语句可以把 `CASE` 表达式写在 `SUM` 函数里面从而简化一下语句

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
    FROM (
        SELECT *, SUM(CASE WHEN content = prev_content THEN 0 ELSE 1 END) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
        FROM (
            SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
            FROM message
        ) AS preved
    ) AS grouped
) AS counted
WHERE group_count > 1
ORDER BY id;
```

这种方法可以通过调节每个分组中的记录数即可实现消息必须连续出现任意次数的需求，比如改为 `group_count > 2`，就可以实现消息必须连续出现三次及以上的需求。

### 使用全局行号和局部行号的差作为分组的 ID

```sql
WITH rowed AS (
    SELECT *,
           ROW_NUMBER() OVER(PARTITION BY sender ORDER BY create_at) AS global_row_num, -- 全局行号
           ROW_NUMBER() OVER(PARTITION BY sender, content ORDER BY create_at) AS local_row_num -- 局部行号
    FROM message
),
grouped AS (
    SELECT *, global_row_num - local_row_num AS group_id
    FROM rowed
)
SELECT * FROM grouped ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+--------------+-------------+--------+
|id|sender|content|create_at          |global_row_num|local_row_num|group_id|
+--+------+-------+-------------------+--------------+-------------+--------+
|1 |Bob   |A      |2024-11-09 11:11:11|1             |1            |0       |
|2 |Bob   |A      |2024-11-09 11:11:12|2             |2            |0       |
|3 |Bob   |B      |2024-11-09 11:11:13|3             |1            |2       |
|4 |Bob   |B      |2024-11-09 11:11:14|4             |2            |2       |
|5 |Bob   |B      |2024-11-09 11:11:15|5             |3            |2       |
|6 |Bob   |C      |2024-11-09 11:11:16|6             |1            |5       |
|7 |Bob   |A      |2024-11-09 11:11:17|7             |3            |4       |
|8 |Bob   |A      |2024-11-09 11:11:18|8             |4            |4       |
|9 |Bob   |A      |2024-11-09 11:11:19|9             |5            |4       |
|10|Bob   |D      |2024-11-09 11:11:20|10            |1            |9       |
|11|Bob   |D      |2024-11-09 11:11:21|11            |2            |9       |
|12|Bob   |D      |2024-11-09 11:11:22|12            |3            |9       |
|13|Bob   |E      |2024-11-09 11:11:23|13            |1            |12      |
|14|Alice |A      |2024-11-09 11:11:24|1             |1            |0       |
|15|Alice |A      |2024-11-09 11:11:25|2             |2            |0       |
|16|Alice |B      |2024-11-09 11:11:26|3             |1            |2       |
|17|Alice |A      |2024-11-09 11:11:27|4             |3            |1       |
|18|Alice |A      |2024-11-09 11:11:28|5             |4            |1       |
|19|Alice |C      |2024-11-09 11:11:29|6             |1            |5       |
|20|Alice |C      |2024-11-09 11:11:30|7             |2            |5       |
|21|Alice |D      |2024-11-09 11:11:31|8             |1            |7       |
|22|Alice |D      |2024-11-09 11:11:32|9             |2            |7       |
+--+------+-------+-------------------+--------------+-------------+--------+
```

这种方式的一个问题是在同一大分区内 `global_row_num - local_row_num` 的结果后面的是否会和前面的重复，比如第 6～8 行的结果为 2，在若干行后，比如在第 14～15 行会不会也出现结果为 2。答案是不会的！下面是 ChatGPT 提供的证明结果

{% asset_img mathematical-proof-1.png %}

{% asset_img mathematical-proof-2.png %}

在上面查询语句的基础上使用 `COUNT` 窗口函数并筛选数量大于 1 的记录即可

```sql
WITH rowed AS (
    SELECT *,
           ROW_NUMBER() OVER(PARTITION BY sender ORDER BY create_at) AS global_row_num, -- 全局行号
           ROW_NUMBER() OVER(PARTITION BY sender, content ORDER BY create_at) AS local_row_num -- 局部行号
    FROM message
),
grouped AS (
    SELECT *, global_row_num - local_row_num AS group_id
    FROM rowed
),
counted AS (
    SELECT *, COUNT(*) OVER(partition by sender, group_id) AS group_count
    FROM grouped
)
SELECT * FROM counted WHERE group_count > 1 ORDER BY id;
```

除此之外还可以不使用 `COUNT` 窗口函数，只使用普通的 `COUNT` 函数并使用 `JOIN` 的方式也可以实现相同的结果

```sql
WITH grouped AS (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY sender ORDER BY create_at) - ROW_NUMBER() OVER(PARTITION BY sender, content ORDER BY create_at) AS group_id
    FROM message
),
counted AS (
    -- 计算连续记录的出现次数
    SELECT sender, content, group_id, COUNT(*) AS consecutive_count
    FROM grouped
    GROUP BY sender, content, group_id
    HAVING COUNT(*) > 1 -- 筛选出连续出现两次及以上的记录
)
SELECT m.id, m.sender, m.content, m.create_at
FROM message AS m
    JOIN counted AS c ON m.sender = c.sender AND m.content = c.content
    JOIN grouped AS g ON m.id = g.id AND g.group_id = c.group_id
ORDER BY m.id;
```
