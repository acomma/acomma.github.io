---
title: PostgreSQL TIMESTAMP 类型映射为 Java Date 类型时精度丢失问题
date: 2024-04-27 22:00:33
updated: 2024-04-27 22:00:33
tags: [PostgreSQL, Java]
---

## 问题

在我们的业务中有一个生产者程序不停地向 `message` 表写入记录，简化的 `message` 表的结构如下所示

```sql
CREATE TABLE message
(
    id          SERIAL,      -- ID
    content     VARCHAR,     -- 消息内容
    create_time TIMESTAMPTZ, -- 创建时间
    PRIMARY KEY (id)
);
```

另外有一个消费者程序不停地读取 `message` 表的数据，这个消费者程序依赖一个称为“水位线”的时间戳，所有创建时间小于等于水位线的消息都已经处理，我们使用 `consumer` 表来记录当前的水位线

```sql
CREATE TABLE consumer
(
    id        SERIAL,      -- ID
    watermark TIMESTAMPTZ, -- 水位线
    PRIMARY KEY (id)
);
```

我们会有一个定时任务每隔一段时间调度一次消费者程序，消费者程序的主要逻辑是

1. 检查当前水位线

    ```shell
    test=# SELECT watermark FROM consumer;
           watermark
    ------------------------
     2024-04-30 15:16:17+08
    (1 row)
    ```

2. 使用当前水位线加载待处理消息

    ```sql
    test=# SELECT content, create_time FROM message WHERE create_time > '2024-04-30 15:16:17+08' AND create_time <= NOW() ORDER BY create_time;
    content |          create_time
    ---------+-------------------------------
    aaa     | 2024-04-30 16:17:18.936795+08
    bbb     | 2024-04-30 17:18:19.956867+08
    (2 rows)
    ```

3. 逐条处理消息，并用当前消息的创建时间更新水位线，最后一条消息处理完成时

    ```sql
    test=# UPDATE consumer SET watermark = '2024-04-30 17:18:19.956867+08';
    UPDATE 1
    test=# SELECT watermark FROM consumer;
            watermark
    -------------------------------
    2024-04-30 17:18:19.956867+08
    (1 row)
    ```

就当前的 SQL 版本模拟的逻辑而言在定时任务第 2 轮调度时将没有待处理消息。但是当我们使用 Java 语言实现上面的逻辑时我们发现在定时任务第 2 轮调度时内容为 `bbb` 的消息会再一次被处理。在 Java 版本的实现里我们使用了 `java.util.Date` 类型来表示 `create_time` 和 `watermark`。

<!-- more -->

## 分析

通过 Debug 我们发现在 Java 中使用 `java.util.Date` 类型来表示 PostgreSQL 的 `TIMESTAMPTZ` 类型时 Java 的 `createTime` 字段的值和 PostgreSQL 的 `create_time` 字段值不一样

{% asset_img create_time.png %}

以内容为 `bbb` 的记录为例，在 PostgreSQL 中创建时间的值为 `2024-04-30 17:18:19.956867+08`，而 Java 中的创建时间为 `2024-04-30T17:18:19.956+0800`。通过观察发现这两个时间值除了格式上的差异以外最大的不同是在“秒域”后面 Java 的时间值比 PostgreSQL 的时间值少了一部分，PostgreSQL 时间值在秒域后面是 `.956867`，Java 时间值在秒域后面是 `.956`。

当我们在定时任务第 1 轮调度处理完消息后用 Java 时间值作为参数去更新水位线后

```shell
test=# SELECT watermark FROM consumer;
         watermark
----------------------------
 2024-04-30 17:18:19.956+08
(1 row)
```

在定时任务第 2 轮调度时用这个新水位线去查询消息时

```shell
test=# SELECT content, create_time FROM message WHERE create_time > '2024-04-30 17:18:19.956+08' AND create_time <= NOW() ORDER BY create_time;
 content |          create_time
---------+-------------------------------
 bbb     | 2024-04-30 17:18:19.956867+08
(1 row)
```

从这里可以发现时间值 `2024-04-30 17:18:19.956+08` 要比时间值 `2024-04-30 17:18:19.956867+08` 小，也就是说 Java 时间值 `2024-04-30T17:18:19.956+0800` 不等于 PostgreSQL 时间值 `2024-04-30 17:18:19.956867+08`。是什么原因导致了时间值不相等的结果呢？

## 原因

在 PostgreSQL 文档 [8.5. 日期/时间类型](http://www.postgres.cn/docs/12/datatype-datetime.html) 中有这么一段话

> `time`、`timestamp` 和 `interval` 接受一个可选的精度值 `p`，这个精度值声明在秒域中小数点之后保留的位数。

PostgreSQL [8.5.2. 日期/时间输出](http://www.postgres.cn/docs/12/datatype-datetime.html#DATATYPE-DATETIME-OUTPUT)缺省是 ISO 格式，在输出时它采用一个空格而不是 T。结合这两点我们可以确认时间值 `2024-04-30 17:18:19.956867+08` 中 `.956867` 这部分表示的是 `0.956867` 秒。

如果使用 [EXTRACT](http://www.postgres.cn/docs/12/functions-datetime.html#FUNCTIONS-DATETIME-EXTRACT) 函数抽取 `epoch` 域可以得到

```shell
test=# SELECT create_time, EXTRACT(EPOCH FROM create_time) AS create_time_epoch FROM message;
          create_time          | create_time_epoch
-------------------------------+-------------------
 2024-04-30 16:17:18.936795+08 | 1714465038.936795
 2024-04-30 17:18:19.956867+08 | 1714468699.956867
(2 rows)
```

`create_time_epoch` 字段的值是自 `1970-01-01 00:00:00 UTC` 以来的秒数，把它再乘以 `1000` 转换为毫秒看看

```shell
test=# SELECT create_time, EXTRACT(EPOCH FROM create_time) AS create_time_epoch, EXTRACT(EPOCH FROM create_time) * 1000 AS create_time_milliseconds FROM message;
          create_time          | create_time_epoch | create_time_milliseconds
-------------------------------+-------------------+--------------------------
 2024-04-30 16:17:18.936795+08 | 1714465038.936795 |        1714465038936.795
 2024-04-30 17:18:19.956867+08 | 1714468699.956867 |        1714468699956.867
(2 rows)
```

我们发现 `create_time_milliseconds` 字段小数点前面部分和上图中创建时间的 `fastTime` 属性的值一致，而 `java.util.Date` 类型的 `fastTime` 保存的是时间的毫秒值，对比两个值发现 `fastTime` 比 `create_time_milliseconds` 少了小数点后面的部分，即 `.867`，它表示的是 `0.867` 毫秒（`867` 微秒）。

原因基本可以确定了，PostgreSQL 的 `TIMESTAMPTZ` 类型可以保存时间的微秒值，而 Java 的 `java.util.Date` 类型只能保存时间的毫秒值，在把 `TIMESTAMPTZ` 转换为 `java.util.Date` 时丢掉了微秒部分。在哪里丢掉的呢？

我们打开 MyBatis 的 `DateTypeHandler`，查看其中的一个 `getNullableResult` 方法，比如

```java
public Date getNullableResult(ResultSet rs, String columnName) throws SQLException {
    Timestamp sqlTimestamp = rs.getTimestamp(columnName);
    return sqlTimestamp != null ? new Date(sqlTimestamp.getTime()) : null;
}
```

这个方法的逻辑是先从 `ResultSet` 获取 `Timestamp` 对象，再创建 `Date` 对象并返回。而 `Timestamp` 类的注释中有这么一句话

> It adds the ability to hold the SQL TIMESTAMP fractional seconds value, by allowing the specification of fractional seconds to a precision of nanoseconds.

Java 的 `Timestamp` 类型是可以承接 PostgreSQL 的 `TIMESTAMPTZ` 类型的微秒部分的，使用的是 `Timestamp` 的私有属性 `nanos`。我们跟踪 `PgResultSet` 的 `getTimestamp` 方法调用直到 `TimestampUtils#toTimestamp` 方法

```java
public @PolyNull Timestamp toTimestamp(@Nullable Calendar cal,
    @PolyNull String s) throws SQLException {
    try (ResourceLock ignore = lock.obtain()) {
        // 省略一些边界判断逻辑...

        // s 的值为 2024-04-30 17:18:19.956867+08
        ParsedTimestamp ts = parseBackendTimestamp(s);
        Calendar useCal = ts.hasOffset ? getCalendar(ts.offset) : setupCalendar(cal);
        useCal.set(Calendar.ERA, ts.era);
        useCal.set(Calendar.YEAR, ts.year);
        useCal.set(Calendar.MONTH, ts.month - 1);
        useCal.set(Calendar.DAY_OF_MONTH, ts.day);
        useCal.set(Calendar.HOUR_OF_DAY, ts.hour);
        useCal.set(Calendar.MINUTE, ts.minute);
        useCal.set(Calendar.SECOND, ts.second);
        useCal.set(Calendar.MILLISECOND, 0);

        Timestamp result = new Timestamp(useCal.getTimeInMillis());
        // ts.nanos 的值为 956867000
        result.setNanos(ts.nanos);
        return result;
    }
}
```

再来看看 `Timestamp#getTime` 方法的实现

```java
public long getTime() {
    // 获取的是父类 Date 的毫秒时间，1714468699000
    long time = super.getTime();
    // nanos 的值为 956867000，nanos / 1000000 等于 956
    return (time + (nanos / 1000000));
}
```

在创建 `Date` 对象时通过 `nanos / 1000000` 取余丢掉了微秒部分。

## 结论

Java 的 `java.util.Date` 类型只能保存毫秒精度的时间，而 PostgreSQL 的 `TIMESTAMPTZ` 类型可以保存微秒精度的时间，在转换时丢掉了微秒部分，从而导致两个时间不一致。

为了保证精度不丢失可以使用 `java.sql.Timestamp`、`java.time.LocalDateTime`、`java.time.OffsetDateTime` 或者 `java.time.ZonedDateTime` 之一。另外一种解决办法是为 `message` 和 `consumer` 表的时间类型指定精度值 `p` 为 3，即使用 `TIMESTAMPTZ(3)`。
