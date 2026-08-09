---
title: 如何让 Kettle 在每小时的整点时刻执行任务？
date: 2026-08-09 11:33:37
updated: 2026-08-09 11:33:37
tags: [Kettle]
---

## 问题描述

如标题所述我们希望 Kettle 在每个小时的整点时刻执行一次任务，比如在 09:00、10:00、11、00、······ 执行一次任务。Kettle 的作业可以配置每 60 分钟调度一次，如 [PDI 定时任务调度问题](https://acomma.github.io/2024/01/13/the-problem-of-pdi-schedule/)所述，这里配置的 60 分钟是指上一次调度完成后下一次调度要等待 60 分钟。举个例子，在 09:00 调度一次任务，如果任务需要花费 20 分钟完成，那么下一次将在 10:20 开始。因此这种配置方式不能满足我们的需求。

{% asset_img every-60-minute.png %}

<!-- more -->

## 解决方法

```sql
CREATE TABLE watermark
(
    id        SERIAL8,     -- 自增主键
    task_code VARCHAR,     -- 任务编码，任务唯一标识
    last_time TIMESTAMPTZ, -- 水位时间，上次处理到的时间水位
    PRIMARY KEY (id)
);

INSERT INTO watermark(id, task_code, last_time) VALUES (1000, 'AAAAA', '2026-08-08 09:00:00');
```

使用一张“水位表”来记录上次执行任务的整点时刻，Kettle 任务设置为每一分钟调度一次，利用 Kettle “计算表中的记录数”组件判断是否满足执行条件，任务执行完成后更新水位表。Kettle 作业的整体情况如下所示

{% asset_img job-overview.png %}

### 开始节点

{% asset_img start-node.png %}

配置每一分钟调度一次 Kettle 作业。

### 获取当前整点时间

{% asset_img transformer-overview.png %}

这是一个转换，包括“表输入”和“设置变量”两个节点。

{% asset_img table-input.png %}

在“表输入”节点查询当前小时和当前分钟。

```sql
SELECT to_char(date_trunc('hour', now()), 'YYYY-MM-DD HH24:MI:SS') AS curr_hour,
       to_char(now(), 'MI')::INT4 AS curr_minute;
```

{% asset_img set-variables.png %}

“设置变量”节点设置全局变量 `CURR_HOUR` 和 `CURR_MINUTE`。

### 是否满足执行时间

{% asset_img is_time_to_execute.png %}

通过 SQL 脚本查询当前小时大于上一次水位时间且当前分钟在 0~10 内的任务，当数据量等于 1 时才会进行下一步。要求“当前分钟在 0~10 内”是为了避免当前时间距离当前整点太远，也可以去掉这个加强条件。

```sql
SELECT 1
FROM (
    SELECT last_time AS last_hour
    FROM watermark
    WHERE task_code = 'AAAAA'
) AS t
WHERE last_hour < '${CURR_HOUR}'::TIMESTAMPTZ
    AND '${CURR_MINUTE}'::INT4 >= 0
    AND '${CURR_MINUTE}'::INT4 < 10;
```

### 更新水位时间

{% asset_img update_watermark_time.png %}

任务执行完成后使用当前小时更新水位时间。

```sql
UPDATE watermark
SET last_time = '${CURR_HOUR}'
WHERE task_code = 'AAAAA';
```

## 结语

只要任务能在 1 小时内执行完成，基本就可以实现每个整点时刻执行一次任务。当前的配置还是有缺陷，如果任务执行超过 1 小时，比如 90 分钟，可能导致跳过下一个整点时刻。
