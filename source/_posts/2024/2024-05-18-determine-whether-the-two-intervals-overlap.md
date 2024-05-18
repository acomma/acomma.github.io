---
title: 判断两个区间是否重叠
date: 2024-05-18 11:34:49
updated: 2024-05-18 11:34:49
tags:
---

## 问题

我们假设区间的表示为 `[start, end]`，对于两个区间 `A` 和 `B` 如何判断它们是否重叠？

<!-- more -->

## 分析

两个区间的关系总共有 6 种情形，情形 1 至 情形 4 表示区间 `A` 和 `B`有重叠，情形 5 和情形 6 表示区间 `A` 和 `B` 没有重叠

{% asset_img overlap.drawio.svg %}

通过上图可以发现区间 `A` 和 `B` 是否重叠可以通过判断它们的起始和结束点来实现

1. 正向思维：区间 `A` 和 `B` 有重叠时 `max(A.start, B.start) <= min(A.end, B.end)`
2. 逆向思维：区间 `A` 和 `B` 有不重叠时 `A.end < B.start || A.start > B.end`，那么当 `A.end >= B.start && A.start <= B.end` 时区间 `A` 和 `B` 有重叠

我们将这个问题拿去问了一下 ChatGPT，ChatGPT 3.5 的回答如下

{% asset_img chatgpt-35.png %}

ChatGPT 4o 的回答如下

{% asset_img chatgpt-4o.png %}

我们发现无论是 ChatGPT 3.5 模型还是最新的 ChatGPT 4o 模型都给出了类似的结论，同时可以发现 ChatGPT 4o 的回答在内容上更完整。

## 实现

### Java 实现

ChatGPT 给出使用 Python 语言如何判断两个区间是否重叠，在这里我们使用 Java 语言实现相同的逻辑，并且给出正向思维和逆向思维两种判断的实现方式。

```java
public class Interval {
    private final Integer start;
    private final Integer end;

    public Interval(Integer start, Integer end) {
        this.start = start;
        this.end = end;
    }

    // 逆向思维
    public boolean overlap(Interval that) {
        return this.end >= that.start && this.start <= that.end;
    }

    // 正向思维
    public boolean overlap1(Interval that) {
        return Math.max(this.start, that.start) <= Math.min(this.end, that.end);
    }

    // 直观方式
    public boolean overlap2(Interval that) {
        return (that.start <= this.start && this.start <= that.end)
                || (that.start <= this.end && this.end <= that.end)
                || (this.start <= that.start && that.start <= this.end)
                || (this.start <= that.end && that.end <= this.end);
    }
}
```

### PostgreSQL 实现

我们创建一个区间表 `interval` 并且存入一些定义好的区间

```sql
CREATE TABLE interval
(
    id    INTEGER, -- ID
    start INTEGER, -- 起点
    "end" INTEGER, -- 终点
    PRIMARY KEY (id)
);

INSERT INTO interval(id, start, "end") VALUES (1, 2, 5); -- 区间 [2, 5]
INSERT INTO interval(id, start, "end") VALUES (2, 3, 7); -- 区间 [3, 8]
```

现在我们来查询出与区间 `[6, 9]` 重叠的区间。在 PostgreSQL 中可以使用 `GREATEST` 和 `LEAST` 来实现正向思维描述的判断重叠的逻辑

```sql
test=# SELECT * FROM interval WHERE GREATEST(start, 6) <= LEAST("end", 9);
 id | start | end
----+-------+-----
  2 |     3 |   8
(1 row)
```

下面是逆向思维的方法实现相应的 SQL 查询

```sql
test=# SELECT * FROM interval WHERE "end" >= 6 AND start <= 9;
 id | start | end
----+-------+-----
  2 |     3 |   8
(1 row)
```

也可以按照 ChatGPT 4o 中描述的判断端点是否在范围内

```sql
test=# SELECT * FROM interval WHERE (start <= 6 AND 6 <= "end") OR (start <= 9 AND 9 <= "end") OR (6 <= start AND start <= 9) OR (6 <= "end" AND "end" <= 9);
 id | start | end
----+-------+-----
  2 |     3 |   8
(1 row)
```

这种方式虽然也能实现功能，但是它的代码就相对长了很多。这种方式也有一个好处，它的实现逻辑是自然的。

## 总结

使用逆向思维 `A.end >= B.start && A.start <= B.end` 来判断两个区间是否重叠初看起来不是那么直观，但是它的实现最简单，不需要借助其他辅助函数就能完成判断，推荐采用这种方法。

## 参考

1. [(算法)判断两个区间是否重叠](https://www.cnblogs.com/AndyJee/p/4537251.html)
2. [java 判断数值区间是否有重叠](https://blog.csdn.net/qq_43371004/article/details/119643187)
3. [判断两个时间段交集、时间重叠问题](https://juejin.cn/post/7188012060729901114)
4. [SQL查询范围重叠的数据](https://blog.csdn.net/yishengreai/article/details/6865426)
5. [js/java判断两个区间是否存在重叠交叉](https://blog.csdn.net/Mister_SNAIL/article/details/77860240)
6. [SQL查询时间段重合的记录](https://zhuanlan.zhihu.com/p/341767416)
7. [判断两个时间段范围是否有交集](https://blog.csdn.net/HXNLYW/article/details/102701632)
8. [SQL中的时间重叠问题](https://blog.csdn.net/liyue071714118/article/details/121433777)
