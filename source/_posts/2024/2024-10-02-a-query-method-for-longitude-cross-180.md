---
title: 一种解决经度跨越 180° 时的查询方法
date: 2024-10-02 21:47:43
updated: 2024-10-02 21:47:43
tags: [SQL, PostgreSQL]
---

{% asset_img area.png %}

如上图所示，有紫色、红色、黄色三个矩形区域，其中紫色区域完全在东半球，黄色区域完全在西半球，而红色区域一半在东半球一半在西半球，即它跨越了 180°(-180°) 经线。我们用左上/右下两个点来表示一个矩形区域，因此紫色区域表示为 `(147.66, 36.31) / (159.71, 30.88)`，红色区域表示为 `(173.53, 36.49) / (-173.32, 31.17)`，黄色区域表示为 `(-160.49, 36.31) / (-147.24, 30.98)`。在每一个区域中分布着一些点，现在需要一种把这些点查询出来的方法。

<!-- more -->

为了实现这个查询，我们需要创建一张存储这些点的表，其中 P1/P2 在紫色区域，P3/P4/P5 在红色区域，P6/P7/P8/P9 在黄色区域

```sql
CREATE TABLE points
(
    name      VARCHAR, -- 名称
    longitude NUMERIC, -- 经度
    latitude  NUMERIC, -- 维度
    remark    VARCHAR, -- 备注
    PRIMARY KEY (name)
);

INSERT INTO points(name, longitude, latitude, remark) VALUES ('P1', 149.35, 34.87, '紫色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P2', 151.51, 35.23, '紫色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P3', 177.06, 34.96, '红色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P4', 178.83, 32.57, '红色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P5', -177.19, 35.23, '红色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P6', -159.28, 35.59, '黄色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P7', -157.52, 33.87, '黄色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P8', -155.08, 34.87, '黄色');
INSERT INTO points(name, longitude, latitude, remark) VALUES ('P9', -151.55, 34.42, '黄色');
```

要查询每个区域都有哪些点在里面，我们结合左上/右下定义区域的特点使用下面的过滤条件

```sql
SELECT *
FROM points
WHERE 1 = 1
    AND (longitude >= upper_left_longitude AND longitude <= lower_right_longitude)
    AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude);
```

当区域整个都在东半球或西半球时，即紫色区域和黄色区域，我们可以使用 `longitude >= upper_left_longitude AND longitude <= lower_right_longitude` 这样的表达式。但是对于一半在东半球一半在西半球的区域，即红色区域，来说就不能使用这个表达式了。我们可以把红色区域的经度带入表达式看看 `longitude >= 173.53 AND longitude <= -173.32`，此时不会有任何结果。

我们知道经度的范围为 `[-180°, 180°]`，而 180° 经线和 -180° 经线是同一条经线，对于跨 180°(-180°) 经线的区域来说，一个点要么在东半球要么在西半球，当左上/右下两个点的经度的差的绝对值超过 180° 时应该拆分为两个查询 `longitude >= upper_left_longitude AND longitude <= 180`（表示在东半球）和 `longitude >= -180 AND longitude <= lower_right_longitude`（表示在西半球）的组合，即

```sql
-- 表示在东半球
SELECT *
FROM points
WHERE 1 = 1
    AND (longitude >= upper_left_longitude AND longitude <= 180)
    AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude)
UNION ALL
-- 表示在西半球
SELECT *
FROM points
WHERE 1 = 1
    AND (longitude >= -180 AND longitude <= lower_right_longitude)
    AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude);
```

为了方便查询三种区域类型的情况，我们需要把查询写到一个函数里面

```sql
CREATE OR REPLACE FUNCTION fn_get_points(upper_left_longitude NUMERIC, upper_left_latitude NUMERIC, lower_right_longitude NUMERIC, lower_right_latitude NUMERIC)
RETURNS SETOF points
AS $$
BEGIN
    IF ABS(upper_left_longitude - lower_right_longitude) < 180 THEN
        RETURN QUERY
            SELECT *
            FROM points
            WHERE 1 = 1
                AND (longitude >= upper_left_longitude AND longitude <= lower_right_longitude)
                AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude);
    ELSE
        RETURN QUERY
            SELECT *
            FROM points
            WHERE 1 = 1
                AND (longitude >= upper_left_longitude AND longitude <= 180)
                AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude)
            UNION ALL
            SELECT *
            FROM points
            WHERE 1 = 1
                AND (longitude >= -180 AND longitude <= lower_right_longitude)
                AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude);
    END IF;
END;
$$ LANGUAGE plpgsql;
```

现在下面的三个查询都可以得到想要的结果

```sql
-- 紫色
SELECT * FROM fn_get_points(147.66, 36.31, 159.71, 30.88);

-- 红色
SELECT * FROM fn_get_points(173.53, 36.49, -173.32, 31.17);

-- 黄色
SELECT * FROM fn_get_points(-160.49, 36.31, -147.24, 30.98);
```

上面的函数还是有点复杂，可以使用[在 PostgreSQL 函数中使用可选条件](https://acomma.github.io/2024/08/31/optional-condition-in-postgresql-function/)中的方法简化 `IF ... ELSE ... END IF` 表达式。而表达一个点要么在东半球要么在西半球还可以使用 `(longitude >= upper_left_longitude AND longitude <= 180) OR (longitude >= -180 AND longitude <= lower_right_longitude)` 简化 `UNION ALL` 查询

```sql
CREATE OR REPLACE FUNCTION fn_get_points(upper_left_longitude NUMERIC, upper_left_latitude NUMERIC, lower_right_longitude NUMERIC, lower_right_latitude NUMERIC)
RETURNS SETOF points
AS $$
BEGIN
    RETURN QUERY
        SELECT *
        FROM points
        WHERE 1 = 1
            AND (
                (ABS(upper_left_longitude - lower_right_longitude) < 180 AND (longitude >= upper_left_longitude AND longitude <= lower_right_longitude))
                OR
                (ABS(upper_left_longitude - lower_right_longitude) >= 180 AND ((longitude >= upper_left_longitude AND longitude <= 180) OR (longitude >= -180 AND longitude <= lower_right_longitude)))
            )
            AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude);
END;
$$ LANGUAGE plpgsql;
```

因为在跨 180°(-180°) 经线时 `(longitude >= upper_left_longitude AND longitude <= 180) OR (longitude >= -180 AND longitude <= lower_right_longitude)` 与 `longitude >= upper_left_longitude OR longitude <= lower_right_longitude` 又是等价的，因此可以进一步简化为

```sql
CREATE OR REPLACE FUNCTION fn_get_points(upper_left_longitude NUMERIC, upper_left_latitude NUMERIC, lower_right_longitude NUMERIC, lower_right_latitude NUMERIC)
RETURNS SETOF points
AS $$
BEGIN
    RETURN QUERY
        SELECT *
        FROM points
        WHERE 1 = 1
            AND (
                (ABS(upper_left_longitude - lower_right_longitude) < 180 AND (longitude >= upper_left_longitude AND longitude <= lower_right_longitude))
                OR
                (ABS(upper_left_longitude - lower_right_longitude) >= 180 AND (longitude >= upper_left_longitude OR longitude <= lower_right_longitude))
            )
            AND (latitude >= lower_right_latitude AND latitude <= upper_left_latitude);
END;
$$ LANGUAGE plpgsql;
```

这种简化其实是利用了经度的特点，没有经度大于 180° 的点，也没有经度小于 -180° 的点，当经度大于 180° 或者小于 -180° 时可以用另一个半球的经度来表示这个点。

完～
