---
title: Gaps and Islands：间隙与连续区间分组方法详解
date: 2026-10-26 21:40:00
updated: 2026-10-26 21:40:00
tags: [SQL, PostgreSQL]
description: 从连续相同值、连续编号到日期与重叠区间，系统讲解 Gaps and Islands 的建模、窗口函数算法、间隙计算与常见陷阱，并复盘一篇 PostgreSQL 连续消息查询。
---

说明：这是一篇由 DeepSeek Harness + DeepSeek-V4.1-Flash 1017 完成的文章，问题来自使用 GitHub Copilot + GPT 5.6 进行实际业务开发时出现在回答中的 Gaps and Islands（间隙与连续区间分组方法）关键字。我通读了整篇文章，只让 DSH 补充了两张说明图，文章中的示例 SQL 没有进行验证。余下内容为正文部分。

只要数据有顺序，就迟早会遇到两个问题：哪些记录连成了一段？段与段之间，又少了什么？

前一个问题叫 **Islands（连续区间/岛屿）**，后一个问题叫 **Gaps（间隙）**。它们不是某个数据库的内置函数，而是一类把“顺序问题”转化为“分组问题”的建模方法。本文以 PostgreSQL 语法为例，但核心模式同样适用于 SQL Server、Oracle、MySQL 8+ 和 SQLite 3.25+。

<!-- more -->

## 问题：连续段与缺失段

### 两个定义

给定一个确定的排序和一个判定条件，可以这样描述：

- **岛屿**：满足判定条件的**极大**连续行集合。“极大”意味着不能再向左或向右扩展一行，否则就会破坏连续性。
- **间隙**：两个相邻岛屿之间，按业务定义“本应存在”但数据中缺失的键范围。

这两个定义里有两个容易被忽略的词：**排序**和**业务定义**。

SQL 表是集合，不是文件，物理存储顺序没有语义。说两行“相邻”，必须先有 `ORDER BY`。同样，间隙也不是凭空产生的：整数序列的间隙是缺失的整数，日期序列的间隙是缺失的日期，区间覆盖的间隙是未被任何区间覆盖的时间段。域不同，答案就不同。

{% asset_img gaps-and-islands-overview.svg %}

图 1 展示了一条已经排序的 `ticket_no` 序列。`1001`、`1002`、`1003` 构成第一个岛；`1007`、`1008` 构成第二个岛；`1012` 单独构成第三个岛。岛与岛之间的 `1004` 到 `1006`、`1009` 到 `1011`，就是间隙。

### 为什么不能直接 GROUP BY

最直观的想法是：按内容分组，再数一数每组有多少行。但它会把“不连续但值相同”的记录混在一起。

{% asset_img group-by-vs-islands.svg %}

图 2 对比了两种分组：直接按 `content` 聚合，只会得到 `A = 4`、`B = 1` 两个值分组；A 的四行其实被 B 分成了两个连续段。按连续段分组则得到三个岛：`A×2`、`B×1`、`A×2`。**值相同不等于连续，连续是相对于顺序而言的。**

所以，Gaps and Islands 的关键不是 `GROUP BY` 本身，而是先构造一个额外的分组键：**同一个岛内保持不变，跨过岛边界时改变**。有了这个键，后面的 `GROUP BY`、`COUNT`、`MIN/MAX` 都只是普通聚合。

### 三种常见形态

| 形态 | “连续”的含义 | 典型例子 |
| --- | --- | --- |
| 连续相同值 | 相邻行的业务值相同 | 连续登录状态、连续告警级别、连续消息内容 |
| 连续编号/日期 | 键值按固定步长递增 | 发票号、流水号、连续打卡日 |
| 连续覆盖区间 | 区间相交或首尾相接 | 房间占用、设备在线时段、优惠活动时间 |

这三种形态对应不同的算法。把它们混为一谈，是很多错误查询的源头。

> 术语说明：不同资料对 Gaps and Islands 的边界略有差异。最狭义的版本只讨论连续编号/日期；本文把连续相同值的 run（游程，即连续相同值组成的一段，例如 `A A` 或 `B B B`）也视为 island 问题，因为它们的核心都是“按顺序切分极大连续段”。在评估旧文时，这个区分很重要。

### 先决定你要哪种结果

同一个问题，至少有三种输出形态：

1. **成员明细**：保留原始行，并附加一个 `island_id`。
2. **岛屿摘要**：每个岛一行，包含 `start`、`end`、`COUNT(*)`。
3. **间隙列表**：每段缺失范围一行，包含 `gap_start`、`gap_end`。

目标不同，SQL 的最后一步也不同。不要一开始就写窗口函数，先确定输出形态。

## 统一思路：让岛内共享一个键

所有 Gaps and Islands 查询都可以拆成同一条流水线：

1. **定义顺序**：确定 `ORDER BY`，并保证它唯一、稳定。
2. **标记边界**：判断当前行是否开启了一个新岛。
3. **生成岛号**：把边界标记累加，或利用行号差分，得到分组键。
4. **分组聚合**：按岛号汇总，得到起止、长度，或保留成员。
5. **求间隙**：对岛屿摘要使用 `LEAD`，或对补全后的域做反连接。

不同方法的差别，只在第 2、3 步如何实现：

| 方法 | 适合的形态 | 岛键怎么来 | 主要限制 |
| --- | --- | --- | --- |
| 值减行号 | 连续编号/日期 | `value - ROW_NUMBER()` | 值必须按固定步长递增；重复值要先处理 |
| 双行号差分 | 连续相同值 | `ROW_NUMBER() - ROW_NUMBER(PARTITION BY value)` | 键只在同一值内唯一，分组时必须带上值 |
| `LAG` + 累计和 | 连续相同值、任意边界条件 | `SUM(is_new) OVER (...)` | 需要正确处理首行、空值和窗口帧 |
| 排序 + 历史最远右端点 | 重叠/相接区间 | `MAX(end) OVER (...)` + 累计和 | 要明确开闭区间语义 |

记住一句话：**Gaps and Islands 的核心产物是分组键，不是某个窗口函数。**

## 连续编号：最经典的行号差分

先看最简单的形态：一组按固定步长递增的编号，中间少了一些值。

```sql
CREATE TABLE ticket (
    ticket_no integer PRIMARY KEY
);

INSERT INTO ticket (ticket_no) VALUES
    (1001), (1002), (1003), (1007), (1008), (1012);
```

### 生成岛号

```sql
WITH numbered AS (
    SELECT
        ticket_no,
        ticket_no - ROW_NUMBER() OVER (ORDER BY ticket_no) AS grp
    FROM ticket
)
SELECT
    MIN(ticket_no) AS island_start,
    MAX(ticket_no) AS island_end,
    COUNT(*)       AS island_size,
    grp
FROM numbered
GROUP BY grp
ORDER BY island_start;
```

{% asset_img row-number-difference.svg %}

在同一个连续段内，`ticket_no` 每行加 1，`ROW_NUMBER()` 也每行加 1，两者的差不变。跨过间隙时，`ticket_no` 跳过了一段，而行号只加 1，差值就会改变。图 3 把这层算术关系直接摊开在了表格里。

查询结果：

| island_start | island_end | island_size | grp |
| --- | --- | --- | --- |
| 1001 | 1003 | 3 | 1000 |
| 1007 | 1008 | 2 | 1003 |
| 1012 | 1012 | 1 | 1006 |

`grp` 的具体数值没有业务含义，它可以是 1000、1003、1006，也可以是别的。只要满足“同岛相同、异岛不同”，它就是一个合格的岛键。

### 求间隙

有了岛屿摘要，间隙就是相邻两个岛之间的补集。用 `LEAD` 取出下一个岛的开始，再和当前岛的结束比较：

```sql
WITH numbered AS (
    SELECT
        ticket_no,
        ticket_no - ROW_NUMBER() OVER (ORDER BY ticket_no) AS grp
    FROM ticket
),
islands AS (
    SELECT
        MIN(ticket_no) AS island_start,
        MAX(ticket_no) AS island_end
    FROM numbered
    GROUP BY grp
)
SELECT
    island_end + 1 AS gap_start,
    next_start - 1 AS gap_end
FROM (
    SELECT
        *,
        LEAD(island_start) OVER (ORDER BY island_start) AS next_start
    FROM islands
) AS t
WHERE next_start > island_end + 1
ORDER BY gap_start;
```

结果：

| gap_start | gap_end |
| --- | --- |
| 1004 | 1006 |
| 1009 | 1011 |

这里只返回“两个岛之间”的间隙。第一个岛之前、最后一个岛之后是否有间隙，取决于你定义的域边界。如果业务域从 1000 开始、到 1020 结束，就需要额外的边界行或单独的判断，不能指望 `LEAD` 自动补出两端。

### 重复值怎么办

上面的写法默认 `ticket_no` 唯一。如果同一个编号可能出现多次，`ROW_NUMBER()` 会在重复行上继续递增，差值就不再稳定。此时可以先去重：

```sql
SELECT DISTINCT ticket_no
FROM ticket;
```

也可以改用 `DENSE_RANK()`：

```sql
SELECT
    ticket_no,
    ticket_no - DENSE_RANK() OVER (ORDER BY ticket_no) AS grp
FROM ticket;
```

`DENSE_RANK()` 对相同的值给出相同排名，所以重复编号不会破坏岛键。但要注意：这相当于把“同一个编号的多次出现”视为同一个序列值。如果它们代表不同业务实体，应该先想清楚“连续”到底按什么键定义，而不是直接去重。

## 连续相同值：两种等价但气质不同的写法

现在换一种问题：不是编号递增，而是相邻行的状态相同。例如下面这张事件表：

```sql
CREATE TABLE event (
    id     integer PRIMARY KEY,
    status text NOT NULL
);

INSERT INTO event (id, status) VALUES
    (1, 'A'), (2, 'A'),
    (3, 'B'), (4, 'B'), (5, 'B'),
    (6, 'A'), (7, 'A'),
    (8, 'C');
```

期望得到四个岛：`A(1,2)`、`B(3,5)`、`A(6,7)`、`C(8,8)`。注意第一个 A 和第三个 A 虽然值相同，但被 B 隔开，必须是两个不同的岛。

### 方法一：LAG 标记边界，再累计求和

```sql
WITH marked AS (
    SELECT
        id,
        status,
        CASE
            WHEN status IS DISTINCT FROM LAG(status) OVER (ORDER BY id)
            THEN 1
            ELSE 0
        END AS is_new_island
    FROM event
),
grouped AS (
    SELECT
        *,
        SUM(is_new_island) OVER (
            ORDER BY id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS island_id
    FROM marked
)
SELECT
    island_id,
    status,
    MIN(id)  AS first_id,
    MAX(id)  AS last_id,
    COUNT(*) AS island_size
FROM grouped
GROUP BY island_id, status
ORDER BY first_id;
```

{% asset_img lag-running-sum.svg %}

图 4 展示了这个过程：第一行没有前驱，记为新岛；之后每次值发生变化，`is_new` 为 1；把 `is_new` 从上往下累加，就得到了岛号。同一个岛号的行自然可以 `GROUP BY`。

结果：

| island_id | status | first_id | last_id | island_size |
| --- | --- | --- | --- | --- |
| 1 | A | 1 | 2 | 2 |
| 2 | B | 3 | 5 | 3 |
| 3 | A | 6 | 7 | 2 |
| 4 | C | 8 | 8 | 1 |

这段查询有几个细节值得说明。

**第一，为什么用 `IS DISTINCT FROM`？** 普通等号在遇到 `NULL` 时返回 `UNKNOWN`，`CASE WHEN ... THEN 1 ELSE 0` 会把它导向 `ELSE`，于是每个 `NULL` 都被当成“新岛”。如果业务上连续的 `NULL` 也应该算同一段，就必须使用空值安全的比较。本文的 `status` 是 `NOT NULL`，但写成 `IS DISTINCT FROM` 仍然是更稳妥的习惯。

**第二，为什么显式写 `ROWS`？** PostgreSQL 中，带有 `ORDER BY` 的窗口聚合默认帧是 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`。如果排序键有重复值，同一序值的多行会一起进入当前帧，累计和可能被“成批”计算。显式写 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，语义才是逐行累加。更根本的解决办法，还是让 `ORDER BY` 唯一。

**第三，首行为什么是 1？** 本节的 `status` 是 `NOT NULL`，第一行的 `LAG` 返回 `NULL`，`status IS DISTINCT FROM NULL` 为真，所以它被视为新岛的开始。这符合直觉：第一行当然是某个岛的第一行。

如果比较列本身允许 `NULL`，这里有一个小细节：首行的 `LAG` 是 `NULL`，当前值也是 `NULL` 时，`NULL IS DISTINCT FROM NULL` 为假，首个 `NULL` 岛会得到 `island_id = 0`。分组结果仍然正确（所有连续的 `NULL` 仍在同一组），只是编号从 0 开始。如果要求编号一定从 1 开始，可以在条件里显式加上“当前行是分区第一行”，或者改用双行号差分。

### 方法二：双行号差分

另一种写法不依赖 `LAG` 和累计和，而是用两个 `ROW_NUMBER()` 相减：

```sql
WITH numbered AS (
    SELECT
        id,
        status,
        ROW_NUMBER() OVER (ORDER BY id) AS rn_all,
        ROW_NUMBER() OVER (
            PARTITION BY status ORDER BY id
        ) AS rn_status
    FROM event
)
SELECT
    status,
    MIN(id)  AS first_id,
    MAX(id)  AS last_id,
    COUNT(*) AS island_size,
    rn_all - rn_status AS island_key
FROM numbered
GROUP BY status, rn_all - rn_status
ORDER BY first_id;
```

{% asset_img double-row-number.svg %}

图 5 把两个行号逐行摊开：全局行号每行加 1，同值行号只在同一个 `status` 内加 1；两者相减，同一个岛内保持不变，跨过边界时才改变。

结果中的 `island_key` 分别是 0、2、3、7：

| status | first_id | last_id | island_size | island_key |
| --- | --- | --- | --- | --- |
| A | 1 | 2 | 2 | 0 |
| B | 3 | 5 | 3 | 2 |
| A | 6 | 7 | 2 | 3 |
| C | 8 | 8 | 1 | 7 |

为什么它有效？对某一行来说，`rn_all` 是它在整个序列中的位置，`rn_status` 是它在“同值分区”中的位置。两者相减，等于“到当前行为止，出现了多少个与当前值不同的行”。在同一个连续段内，没有新的异值行，这个数不变；对于同一个 `status` 的不同连续段，异值行只会更多，所以键不会重复。注意：这个数在不同 `status` 之间并不保证变化，`A B A` 就是反例——B 和第二个 A 的键都是 1。

但这里有一个非常重要的限制：

> `rn_all - rn_status` **只对同一个 `status` 唯一**，对不同 `status` 可能碰撞。

看一个反例：

```text
序列：A  B  A
```

| id | status | rn_all | rn_status | island_key |
| --- | --- | --- | --- | --- |
| 1 | A | 1 | 1 | 0 |
| 2 | B | 2 | 1 | 1 |
| 3 | A | 3 | 2 | 1 |

B 和第二个 A 的 `island_key` 都是 1。如果只按 `island_key` 分组，就会把 B 和后面的 A 合并成一个岛。因此，**分组时必须带上 `status`**，即 `GROUP BY status, island_key`。这个细节会在后面的案例复盘中再次出现。

### 两种方法怎么选

| 维度 | `LAG` + 累计和 | 双行号差分 |
| --- | --- | --- |
| 可读性 | 边界语义直观，容易解释 | 需要理解“异值计数”的含义 |
| 边界条件 | 可以表达任意条件，例如“间隔超过 3 天” | 更适合“值相等”这种简单条件 |
| 空值 | `IS DISTINCT FROM` 可空值安全 | `PARTITION BY` 天然把 `NULL` 放在一起 |
| 岛键 | 连续递增，便于阅读 | 可能跳跃，且只在同值内唯一 |
| 性能 | 一次排序 + 一次窗口 | 两次窗口，通常需要两次排序或一次增量排序 |

如果只是“连续相同值”，两种都可以；如果边界条件更复杂（例如“同一天、同一状态，且间隔不超过 10 分钟”），`LAG` + 累计和的表达力更好。无论选哪种，**排序键唯一**都是前提。

## 日期：连续的是“日历”，不是行号

日期问题比编号问题多一层陷阱：同一天可能有多条记录，而“连续”通常是按天判断的。先看数据：

```sql
CREATE TABLE attendance (
    user_id text NOT NULL,
    day     date NOT NULL
);

INSERT INTO attendance (user_id, day) VALUES
    ('Bob', DATE '2026-10-01'),
    ('Bob', DATE '2026-10-01'),
    ('Bob', DATE '2026-10-02'),
    ('Bob', DATE '2026-10-03'),
    ('Bob', DATE '2026-10-06'),
    ('Bob', DATE '2026-10-07'),
    ('Bob', DATE '2026-10-10');
```

Bob 在 10 月 1 日有两条记录。如果直接对原始行做 `ROW_NUMBER()`，同一天的两行会得到不同的行号，`day - rn` 就会把 10 月 1 日拆开。所以第一步必须按天去重：

```sql
WITH present_days AS (
    SELECT DISTINCT user_id, day
    FROM attendance
),
numbered AS (
    SELECT
        user_id,
        day,
        day - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY day))::int AS grp
    FROM present_days
),
islands AS (
    SELECT
        user_id,
        MIN(day) AS start_day,
        MAX(day) AS end_day,
        COUNT(*) AS days
    FROM numbered
    GROUP BY user_id, grp
)
SELECT *
FROM islands
ORDER BY user_id, start_day;
```

{% asset_img date-islands-gaps.svg %}

结果与图 6 一致：

| user_id | start_day | end_day | days |
| --- | --- | --- | --- |
| Bob | 2026-10-01 | 2026-10-03 | 3 |
| Bob | 2026-10-06 | 2026-10-07 | 2 |
| Bob | 2026-10-10 | 2026-10-10 | 1 |

这里 `day - rn` 的结果是一个日期，而不是整数。PostgreSQL 的 `date - integer` 返回 `date`，`date - date` 才返回天数。`ROW_NUMBER()` 返回 `bigint`，所以代码里显式加了 `::int`。如果不想依赖这个类型转换，也可以先算“距某个基准日期的天数”，再减行号。

### 求日期间隙

```sql
WITH present_days AS (
    SELECT DISTINCT user_id, day
    FROM attendance
),
numbered AS (
    SELECT
        user_id,
        day,
        day - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY day))::int AS grp
    FROM present_days
),
islands AS (
    SELECT
        user_id,
        MIN(day) AS start_day,
        MAX(day) AS end_day
    FROM numbered
    GROUP BY user_id, grp
)
SELECT
    user_id,
    end_day + 1 AS gap_start,
    next_start - 1 AS gap_end
FROM (
    SELECT
        *,
        LEAD(start_day) OVER (
            PARTITION BY user_id ORDER BY start_day
        ) AS next_start
    FROM islands
) AS t
WHERE next_start > end_day + 1
ORDER BY user_id, gap_start;
```

结果：

| user_id | gap_start | gap_end |
| --- | --- | --- |
| Bob | 2026-10-04 | 2026-10-05 |
| Bob | 2026-10-08 | 2026-10-09 |

这里的 `end_day + 1` 和 `next_start - 1` 依赖 PostgreSQL 的日期加减法。换数据库时，要改成对应的 `DATE_ADD`、`DATEADD` 或 `+ INTERVAL '1 day'`。

### 日历补全：另一种“找缺失”的思路

`LEAD` 的方法只返回“观测到的岛之间”的间隙。如果业务域是固定的，例如“10 月 1 日到 10 月 10 日每天都应该打卡”，也可以先构造完整日历，再反连接：

```sql
WITH calendar AS (
    SELECT DATE '2026-10-01' + off AS day
    FROM generate_series(0, 9) AS gs(off)
),
present AS (
    SELECT DISTINCT day
    FROM attendance
    WHERE user_id = 'Bob'
)
SELECT c.day
FROM calendar c
LEFT JOIN present p USING (day)
WHERE p.day IS NULL
ORDER BY c.day;
```

这条查询返回 10 月 4 日、5 日、8 日、9 日，是逐日列表。如果要再合并成连续区间，可以对结果再做一次行号差分。日历补全的优点是能发现首尾缺失；缺点是必须先知道完整的域，而且 `generate_series` 在大范围上会产生大量行。

### “连续”的域由业务决定

日期问题最容易被忽略的，是“连续”到底按什么日历算：

- 按自然日连续：上面的写法即可。
- 按工作日连续：先构造工作日序列，再对工作日序列做行号差分；周末不应该被算成 gap。
- 按月连续：先把日期截断到月初，再用月份序号做差分，不能直接用“加 1 天”。
- 按小时连续：同一天可能有多个小时，先去重到小时，再处理时区与夏令时。

**先定义域，再写 SQL。** 否则算法越漂亮，答案越可能不是业务想要的。

## 重叠区间：排序 + 扫描线

第三种形态不是“值相等”，而是“区间相交”。例如房间占用记录：

```sql
CREATE TABLE span (
    id       integer PRIMARY KEY,
    start_at integer NOT NULL,
    end_at   integer NOT NULL,
    CHECK (start_at < end_at)
);

INSERT INTO span (id, start_at, end_at) VALUES
    (1, 1, 4),
    (2, 3, 6),
    (3, 7, 9),
    (4, 8, 12),
    (5, 14, 15);
```

这里用整数表示半开区间 `[start_at, end_at)`，方便演示。目标是把它合并成尽量少的连续覆盖区间。

{% asset_img interval-merge.svg %}

图 7 中，A 和 B 相交，合并为 `[1, 6)`；C 和 D 相交，合并为 `[7, 12)`；E 独立，保留为 `[14, 15)`。`[6, 7)` 和 `[12, 14)` 是未被覆盖的间隙。

### 为什么不能只看前一行的 end

一种常见的错误写法是：按 `start_at` 排序，然后用 `LAG(end_at)` 判断当前区间是否和上一个区间重叠。这个写法在区间嵌套时会失败。例如：

```text
[1, 10)
   [2, 3)
        [4, 5)
```

`[4, 5)` 和它前一行的 `[2, 3)` 不相交，但它仍然在 `[1, 10)` 的覆盖范围内，应该属于同一个岛。正确做法是维护“已经处理过的所有区间的最大右端点”，而不是只记住前一行的右端点。

### 合并区间

```sql
WITH ordered AS (
    SELECT
        *,
        MAX(end_at) OVER (
            ORDER BY start_at, id
            ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
        ) AS max_end_before
    FROM span
),
marked AS (
    SELECT
        *,
        CASE
            WHEN max_end_before IS NULL OR start_at > max_end_before
            THEN 1
            ELSE 0
        END AS is_new_island
    FROM ordered
),
grouped AS (
    SELECT
        *,
        SUM(is_new_island) OVER (
            ORDER BY start_at, id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS island_id
    FROM marked
)
SELECT
    island_id,
    MIN(start_at) AS start_at,
    MAX(end_at)   AS end_at
FROM grouped
GROUP BY island_id
ORDER BY start_at;
```

结果：

| island_id | start_at | end_at |
| --- | --- | --- |
| 1 | 1 | 6 |
| 2 | 7 | 12 |
| 3 | 14 | 15 |

这段查询有三个关键点：

1. `MAX(end_at) OVER (... 1 PRECEDING)` 只看“当前行之前”的行，得到的是历史最远右端点。
2. 第一行的 `max_end_before` 是 `NULL`，所以它必然开启一个新岛。
3. 判断条件是 `start_at > max_end_before`：对于半开区间，`start_at = max_end_before` 表示首尾相接，仍然连续，应该合并。

### 开区间、闭区间与“相邻”

区间的开闭语义会直接改变合并条件：

| 区间类型 | 相交/相接的判定 | 新岛条件 |
| --- | --- | --- |
| 半开区间 `[start, end)` | `start <= max_end` | `start > max_end` |
| 闭区间 `[start, end]`（连续值） | `start <= max_end` | `start > max_end` |
| 闭区间 `[start, end]`（离散整数，允许相邻） | `start <= max_end + 1` | `start > max_end + 1` |

例如 `[1, 3]` 和 `[4, 6]` 在整数轴上没有共同的点，但也没有缺失的整数，业务上可能希望合并；这时就要用 `start <= max_end + 1`。如果业务认为“必须共享端点才算连续”，则不能用 `+ 1`。

**不要在不说明开闭语义的情况下讨论“重叠”。** 这是区间类查询最常见的歧义。

### 从合并结果求间隙

合并之后，间隙就是相邻两个合并区间之间的空白：

```sql
WITH ordered AS (
    SELECT
        *,
        MAX(end_at) OVER (
            ORDER BY start_at, id
            ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
        ) AS max_end_before
    FROM span
),
marked AS (
    SELECT
        *,
        CASE
            WHEN max_end_before IS NULL OR start_at > max_end_before
            THEN 1
            ELSE 0
        END AS is_new_island
    FROM ordered
),
grouped AS (
    SELECT
        *,
        SUM(is_new_island) OVER (
            ORDER BY start_at, id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS island_id
    FROM marked
),
merged AS (
    SELECT
        island_id,
        MIN(start_at) AS start_at,
        MAX(end_at)   AS end_at
    FROM grouped
    GROUP BY island_id
),
gaps AS (
    SELECT
        end_at AS gap_start,
        LEAD(start_at) OVER (ORDER BY start_at) AS next_start
    FROM merged
)
SELECT
    gap_start,
    next_start AS gap_end
FROM gaps
WHERE next_start > gap_start;
```

结果：

| gap_start | gap_end |
| --- | --- |
| 6 | 7 |
| 12 | 14 |

对于半开区间，间隙是 `[6, 7)` 和 `[12, 14)`。如果下游需要闭区间表示，再根据业务决定是否把端点减 1。

### PostgreSQL 的 range_agg 捷径

如果使用 PostgreSQL 14+，并且区间类型合适，可以把合并交给范围类型：

```sql
SELECT unnest(range_agg(int4range(start_at, end_at))) AS merged
FROM span;
```

`int4range` 默认是 `[start, end)`，`range_agg` 会把重叠或相接的范围合并成一个 `int4multirange`，再用 `unnest` 展开。它很适合做“合并”这一步，但不会自动告诉你间隙，也不改变“开闭语义必须先定义”这件事。跨数据库时，仍然需要上面的可移植写法。

## 结果怎么取：成员、摘要、间隙

到这里，四种算法的“岛键”都已经有了。最后一步是根据需求选择输出形态。

### 保留成员明细

如果下游需要原始行，例如“找出所有属于长度至少为 2 的连续段的记录”，就在分组结果上再加一个窗口计数：

```sql
WITH marked AS (
    SELECT
        id,
        status,
        CASE
            WHEN status IS DISTINCT FROM LAG(status) OVER (ORDER BY id)
            THEN 1
            ELSE 0
        END AS is_new_island
    FROM event
),
grouped AS (
    SELECT
        *,
        SUM(is_new_island) OVER (
            ORDER BY id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS island_id
    FROM marked
),
counted AS (
    SELECT
        *,
        COUNT(*) OVER (PARTITION BY island_id) AS island_size
    FROM grouped
)
SELECT
    id,
    status,
    island_id,
    island_size
FROM counted
WHERE island_size >= 2
ORDER BY id;
```

这种写法不会丢掉原始行，也不会把两个不同的岛合并。`island_id` 只用于分组，不应该被当成业务 ID 或全局唯一 ID。

如果数据里有多个实体（例如多个 `sender` 或多个 `user_id`），`island_id` 只在实体内唯一，计数窗口必须写成 `PARTITION BY sender, island_id`。只写 `PARTITION BY island_id`，会把不同实体中编号相同的岛合并在一起。

### 每个岛一行

如果只需要摘要：

```sql
SELECT
    island_id,
    status,
    MIN(id)  AS first_id,
    MAX(id)  AS last_id,
    COUNT(*) AS island_size
FROM grouped
GROUP BY island_id, status
HAVING COUNT(*) >= 2
ORDER BY first_id;
```

### 间隙列表

对岛屿摘要使用 `LEAD`，就是间隙列表：

```sql
SELECT
    island_end + 1 AS gap_start,
    next_start - 1 AS gap_end
FROM (
    SELECT
        *,
        LEAD(island_start) OVER (ORDER BY island_start) AS next_start
    FROM islands
) AS t
WHERE next_start > island_end + 1;
```

三种输出形态对应三种不同的“最后一步”，不要用同一条 SQL 硬套所有需求。

## 正确性与性能清单

Gaps and Islands 的 SQL 通常不长，但边界条件很多。下面这张表可以作为自查清单：

| 症状 | 常见原因 | 修正 |
| --- | --- | --- |
| 同一个岛被拆成两段 | `ORDER BY` 不唯一，或日期未去重 | 加唯一 tiebreaker；先 `DISTINCT` |
| 两个岛被合并 | 只按值分组，忽略了顺序 | 使用岛键，而不是 `GROUP BY value` |
| 连续 `NULL` 没有被分到一组 | 用 `=` 比较 `NULL` | 改用 `IS DISTINCT FROM` |
| 累计和“成批”跳变 | 默认 `RANGE` 帧遇到重复排序键 | 显式 `ROWS`，并让排序键唯一 |
| 多实体之间互相串组 | 窗口函数缺少 `PARTITION BY` | 所有窗口和分组都带上实体键 |
| 行号差分的键发生碰撞 | 键只在同一值内唯一 | 分组时带上值，或改用累计和 |
| 区间嵌套时合并错误 | 只看 `LAG(end)` | 用历史最大 `MAX(end)` |
| 日期差一整天 | 开闭区间或时区语义不清 | 明确 `[start, end)` 还是 `[start, end]` |

### 排序键必须唯一

这是最重要的一条。`ORDER BY create_at` 在两个事件时间相同时，数据库可以以任意顺序排列它们，`LAG`、`ROW_NUMBER`、`SUM` 的结果都会跟着变。正确的写法是：

```sql
ORDER BY create_at, id
```

把唯一键作为最后的 tiebreaker，窗口函数的结果才是确定的。

### 窗口帧要写清楚

带 `ORDER BY` 的聚合窗口，默认帧是 `RANGE UNBOUNDED PRECEDING`。在排序键唯一时，它和 `ROWS UNBOUNDED PRECEDING` 的结果一样；一旦有重复键，语义就不同。不要依赖“我的数据应该不会重复”这种假设，显式写 `ROWS` 更安全。

### 索引与执行计划

窗口函数通常需要排序。对于按 `sender, create_at` 分区的查询，可以考虑：

```sql
CREATE INDEX ON message (sender, create_at, id);
```

但仍然要用 `EXPLAIN (ANALYZE, BUFFERS)` 确认：

- 是否发生了多次 Sort，尤其是双行号写法；
- 过滤条件是否在窗口计算之前生效；
- 分区键是否与索引前缀匹配；
- 数据量很大时，是否值得把中间结果物化。

Gaps and Islands 的时间复杂度通常由排序主导，约为 `O(n log n)`；窗口函数本身是线性扫描。想靠“加一个索引”解决所有问题并不现实，索引只能帮助排序和过滤，不能改变算法本身。

### 增量与大数据

如果数据是持续追加的，每次都对全表重新计算岛号，成本会越来越高。工程上常见的做法是：

- 只对最近一个时间窗口做增量计算；
- 把已经封闭的岛写入结果表；
- 对“最后一个未封闭的岛”单独维护状态；
- 或者使用流处理中的会话窗口（session window）思路。

这超出了单条 SQL 的范围，但值得在设计阶段考虑。

## 案例复盘：一篇 PostgreSQL 连续消息查询

现在回到那篇 2024 年的文章：[在 PostgreSQL 如何查询连续出现的记录](https://acomma.github.io/2024/11/09/how-to-query-consecutive-record-in-postgresql/)。它的目标是：按 `sender` 分区、按 `create_at` 排序，找出连续出现两次及以上的消息，并保留这些消息的原始行。

这篇文章给出了三种主要写法。把它们的“方法谱系”拆开看，会非常清楚。

### 方法对照

| 原文方法 | 实际技术 | 是否属于 Gaps and Islands | 说明 |
| --- | --- | --- | --- |
| 方法一：`LAG` / `LEAD` 看前后各一行 | 固定偏移的邻行判断 | 否，至少不是通用形式 | 能解决“连续 2 次”；连续 3 次需要更多偏移，不能自然扩展到任意长度 |
| 方法二：`LAG` 标记变化 + `SUM` 累计 | 边界标记 + 累计和 | 是 | 标准 islands 写法，`group_id` 就是岛号，`group_count` 就是岛长 |
| 方法二后半：全局行号 - 局部行号 | 双行号差分 | 是 | 标准 islands 写法，但岛键只在同一 `content` 内唯一 |

所以第一个结论是：**它确实使用了 Gaps and Islands 的方法族，尤其是方法二和双行号差分。** 但它解决的是“islands 筛选”，不是完整的“gaps + islands”：没有计算间隙，也没有把岛压缩成 `start/end` 摘要，而是保留了原始成员行。

### 方法二是标准的“边界 + 累计和”

原文中这段：

```sql
SUM(CASE WHEN content = prev_content THEN 0 ELSE 1 END)
OVER (PARTITION BY sender ORDER BY create_at) AS group_id
```

和前面“连续相同值”一节的 `SUM(is_new_island)` 是同一个模式。`group_id` 在同一个连续段内不变，在内容变化时加 1；再用：

```sql
COUNT(*) OVER (PARTITION BY sender, group_id) AS group_count
```

得到岛长，最后用 `group_count > 1` 筛选。逻辑是完整的，样例数据也能得到正确结果。

### 双行号差分：一个被样例掩盖的键碰撞

原文的第三种写法是：

```sql
ROW_NUMBER() OVER (PARTITION BY sender ORDER BY create_at)
-
ROW_NUMBER() OVER (PARTITION BY sender, content ORDER BY create_at)
AS group_id
```

然后在窗口计数时使用：

```sql
COUNT(*) OVER (PARTITION BY sender, group_id) AS group_count
```

问题就在这里：**`group_id` 只在同一个 `content` 内唯一，不同 `content` 之间可能相同。**

构造一个最小反例：

```text
A  B  A
```

| id | content | rn_all | rn_content | group_id |
| --- | --- | --- | --- | --- |
| 1 | A | 1 | 1 | 0 |
| 2 | B | 2 | 1 | 1 |
| 3 | A | 3 | 2 | 1 |

第二个 A 和 B 的 `group_id` 都是 1。原文的窗口计数只按 `sender, group_id` 分区，于是得到：

| id | content | group_id | group_count |
| --- | --- | --- | --- |
| 1 | A | 0 | 1 |
| 2 | B | 1 | 2 |
| 3 | A | 1 | 2 |

最后 `WHERE group_count > 1` 会返回 B 和第二个 A 两行。但正确答案应该是空：`A`、`B`、`A` 中没有任何连续两次相同的消息。这个反例说明，原文第 446 行的窗口计数少了一个 `content`：

```sql
-- 修正：把 content 加进分区
COUNT(*) OVER (PARTITION BY sender, content, group_id) AS group_count
```

或者不使用窗口计数，改为按 `(sender, content, group_id)` 分组。原文后面的 `JOIN` 版本在 `GROUP BY sender, content, group_id` 中带了 `content`，所以没有这个问题；有问题的只是窗口版本。

这也解释了为什么原文附带的两张“数学证明”截图不可靠。证明中把连续段内的局部行号写成了“从段首重新开始计数”，但 `PARTITION BY content` 并不会在每次内容变化时重置，它统计的是该 `content` 在整个分区内出现的总次数。对于 `A B A`，第二个 A 的局部行号是 2，不是 1；它的 `group_id` 自然可能和 B 相同。**同一个 content 的不同岛，键确实唯一；不同 content 之间，键并不唯一。** 这不是理论吹毛求疵，`A B A` 在实际消息流里很常见。

### 其他边界问题

除了键碰撞，原文还有几处可以加固的地方：

1. **空值比较**：`content = prev_content` 在两个 `NULL` 上返回 `UNKNOWN`，连续的 `NULL` 不会被识别为同一段。如果 `content` 允许为空，应改成 `content IS DISTINCT FROM prev_content`。
2. **排序不稳定**：`ORDER BY create_at` 没有 tiebreaker。如果两条消息的 `create_at` 相同，`LAG` 和 `ROW_NUMBER` 的顺序不确定，连续段可能随执行计划变化。应写成 `ORDER BY create_at, id`。
3. **默认窗口帧**：`SUM(...) OVER (ORDER BY create_at)` 使用的是 `RANGE` 帧，遇到相同 `create_at` 时会把 peer 行一起纳入。显式写 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 更安全。
4. **方法一的扩展性**：`LAG` / `LEAD` 的偏移量是固定的。连续 2 次、3 次可以手写，连续 N 次就需要动态拼接或改用累计和；它适合做“固定长度”的判断，不适合做通用的岛检测。
5. **输出语义**：原文保留的是“属于长度至少为 2 的岛的所有原始行”，而不是“每个岛一行”。这没有问题，但要在业务上明确：结果不是岛摘要，不能直接拿 `group_id` 当唯一岛 ID 使用。

### 一个更稳妥的改写

如果目标仍然是“保留连续出现两次及以上的原始消息”，可以这样写：

```sql
WITH marked AS (
    SELECT
        id,
        sender,
        content,
        create_at,
        CASE
            WHEN content IS DISTINCT FROM LAG(content) OVER (
                PARTITION BY sender
                ORDER BY create_at, id
            )
            THEN 1
            ELSE 0
        END AS is_new_island
    FROM message
),
grouped AS (
    SELECT
        *,
        SUM(is_new_island) OVER (
            PARTITION BY sender
            ORDER BY create_at, id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS island_id
    FROM marked
),
counted AS (
    SELECT
        *,
        COUNT(*) OVER (PARTITION BY sender, island_id) AS island_size
    FROM grouped
)
SELECT
    id,
    sender,
    content,
    create_at
FROM counted
WHERE island_size >= 2
ORDER BY id;
```

这段查询做了四件事：用 `IS DISTINCT FROM` 处理空值；用 `id` 做稳定的 tiebreaker；用显式 `ROWS` 固定累计帧；用 `sender + island_id` 作为唯一岛键。它不依赖“不同 content 的 group_id 不会碰撞”这个错误前提。

### 结论

回到问题本身：**那篇文章是否使用了相同或相似的方法？**

答案是：**使用了相同的方法族，但不是完整的 Gaps and Islands 实现。**

- **相似度：高。** 方法二的“`LAG` 标记变化 + `SUM` 累计”就是标准的 islands 算法；双行号差分也是标准模式。
- **覆盖范围：只做 islands，不做 gaps。** 它关心“哪些消息属于连续段”，不关心“段与段之间缺了什么”，也没有输出岛的起止摘要。
- **正确性：方法二的主路径正确，方法三的窗口版本有一个可复现的键碰撞缺陷。** 样例数据恰好没有触发 `A B A` 这类模式，所以结果看起来没有问题。
- **工程性：需要补上空值、排序 tiebreaker 和显式窗口帧。** 这些不影响核心思想，但会影响边界情况下的结果。

所以，更准确的评价是：**它是一篇“islands-only”的实战文章，方法二已经踩在 Gaps and Islands 的主干上；方法三展示了另一种标准思路，但实现细节需要修正。** 把它称为“连续区间分组”没有问题，称为“完整的 Gaps and Islands 教程”则不够准确。

## 总结

Gaps and Islands 看起来有很多写法，实际上只有三个问题：

1. **顺序是什么？** 没有确定的 `ORDER BY`，就没有“相邻”。
2. **岛的定义是什么？** 是值相等、编号连续、日期连续，还是区间相交？
3. **间隙的域是什么？** 是整数、自然日、工作日，还是连续时间轴？

回答完这三个问题，再选择对应的岛键：

- 连续编号：`value - ROW_NUMBER()`；
- 连续相同值：`LAG` + 累计和，或双行号差分；
- 连续日期：去重后 `date - ROW_NUMBER()`；
- 重叠区间：排序 + 历史最大 `MAX(end)` + 累计和。

最后，记住两个最容易出错的细节：**排序键要唯一，岛键要带上所有能区分岛的业务列。** 前者决定结果是否稳定，后者决定结果是否正确。

本文的图 1 到图 7 均为自绘 SVG，源码与文章放在同一资源目录中。

## 参考资料

- [PostgreSQL Documentation: Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- [PostgreSQL Documentation: Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)
- [PostgreSQL Documentation: Range Functions](https://www.postgresql.org/docs/current/functions-range.html)
- [PostgreSQL Documentation: Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- [Itzik Ben-Gan: Chapter 5, Gaps and Islands (SQL Server MVP Deep Dives)](https://livebook.manning.com/book/sql-server-mvp-deep-dives/chapter-5/ch05table05)
- [Redgate Simple Talk: Gaps and Islands in SQL Server](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/gaps-islands-sql-server-data/)
