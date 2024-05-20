---
title: 计算经纬度所在的时区
date: 2024-05-09 21:15:10
updated: 2024-05-09 21:15:10
tags: [PostgreSQL]
---

## 背景知识

全球共划分为 24 个时区，以本初子午线为基准，从 7.5°W 向东至 7.5°E ，划分为一个时区，叫中时区或零时区。在中时区以东，依次划分为东一区至东十二区；在中时区以西，依次划分为西一区至西十二区。东十二区和西十二区各跨经度 7.5°，合为一个时区。每个时区跨越经度 15°，相邻区域的时间相差 1 小时（这样 24 个时区刚好是 24 小时，地球自转一周）。

{% asset_img timezone.jpg %}

<!-- more -->

## 通用方法

假设以正数 `[0°, 180°]` 表示东经，以负数 `[-0°, -180°]` 表示西经。

根据背景知识第一种计算时区的方式是将经度除以 15，若余数小于 7.5，则除得的商就是该经度所在的时区数；若余数大于 7.5，则该地所在的时区数为商加 1。东经为东时区，西经为西时区。下面是使用 PostgreSQL 函数实现的计算计算经纬度所在的时区的代码

```sql
CREATE OR REPLACE FUNCTION calculate_timezone(longitude NUMERIC) RETURNS NUMERIC AS $$
DECLARE
    quotient NUMERIC;
    remainder NUMERIC;
BEGIN
    quotient = longitude / 15;
    remainder = longitude % 15;
    -- 余数为 0 表示当前经度为时区的中央经线
    IF remainder = 0 THEN
        RETURN FLOOR(quotient);
    END IF;
    -- 余数为 -7.5 或 7.5 表示当前经度为整点时刻所在经线，
    -- 它属于那个时区具有不确定性，这里把它归为前一个时区，
    -- 比如 -22.5° 属于西一区，-7.5° 属于零时区，7.5° 属于零时区，22.5° 属于东一区
    IF -7.5 < remainder AND remainder < 7.5 THEN
        -- 东经
        IF longitude >= 0 THEN
            RETURN FLOOR(quotient);
        END IF;
        -- 西经
        RETURN FLOOR(quotient) + 1;
    ELSE
        -- 东经
        IF longitude >= 0 THEN
            RETURN FLOOR(quotient) + 1;
        END IF;
        -- 西经
        RETURN FLOOR(quotient);
    END IF;
END;
$$ LANGUAGE plpgsql;
```

在 PostgreSQL 中除法运算得到精确的结果，不会对结果进行取整操作，而 `FLOOR` 函数返回不大于参数的最近的整数，因此它可以用来得到除法运算整数类型的商数。余数等于 0 和余数等于 -7.5 或 7.5 的情况已经通过注释进行了说明。西经（经度为负数）时是否加 1 与东经（经度为正数）时是相反的。这种方式计算的时区符合高中地理时区计算逻辑。

第一种方式整体上还是比较复杂的，需要考虑各种边界条件，同时它未考虑到地球自转方向。地球自西向东旋转，东边的地区比西边的地区先看到太阳，即经度大的地方比经度小的地方时间要早一段时间，在国际日期变更线发生日期切换

{% asset_img the-international-date-line.jpeg %}

在这种前提下，-180° 属于西十二区，-172.5° 属于西十一区，-22.5° 属于西一区，-7.5° 属于零时区，7.5° 属于东一区，22.5° 属于东二区，172.5° 属于东十二区，180° 属于东十二区。下面是使用 PostgreSQL 函数实现的计算计算经纬度所在的时区的代码

```sql
CREATE OR REPLACE FUNCTION calculate_timezone(longitude NUMERIC) RETURNS NUMERIC AS $$
BEGIN
    RETURN FLOOR((longitude + 7.5) / 15);
END;
$$ LANGUAGE plpgsql;
```

对 0 时区而言，它的经度范围为 `[-7.5, 7.5]`，经度加上 7.5 后为 `[0, 15]`，它们除以 15 的商数为 `[0, 1]`，应用 `FLOOR` 函数后得到的时区为 0。对东 1 区而言，它的经度范围为 `[7.5, 22.5]`，经度加上 7.5 后为 `[15, 30]`，它们除以 15 的商数为 `[1, 2]`，应用 `FLOOR` 函数后得到的时区为 1。对西 1 区而言，它的经度范围为 `[-22.5, -7.5]`，经度加上 7.5 后为 `[-15, -0]`，它们除以 15 的商数为 `[-1, -0]`，应用 `FLOOR` 函数后得到的时区为 -1。

## 改进方法

通用方法中两种计算方式只使用了经度，它们适用于地球上大多数的地方，但是实际的时区划分还会受到国家或地区和风俗等因素的影响。比如中国，它横跨了 5 个时区，然而中国统一使用东八区作为它的时区。又比如安达曼-尼科巴群岛的时区为东 5.5 时区，完全包含在东六区内。再比如基里巴斯的菲尼克斯群岛莱恩群岛的时区为东十三时区和东十四时区。

{% asset_img TIME_ZONE_16_NO_100_10.webp %}

一种改进的计算方式是同时使用经度和纬度，根据经度和纬度确定坐标所在的区域，用该区域的时区作为最终时区，我们称之为查表法。我们可以从 [ne_10m_time_zones](https://common-data.carto.com/tables/ne_10m_time_zones/public) 下载包含时区与区域对应关系的 CSV 格式的文件 {% asset_link ne_10m_time_zones.csv %}

{% asset_img ne_10m_time_zones.png %}

在 PostgreSQL 中创建一张表 `ne_10m_time_zones`，并把导入下载的 CSV 文件文件中的数据

```sql
CREATE TABLE ne_10m_time_zones
(
    the_geom   VARCHAR,
    objectid   INT4,
    scalerank  INT4,
    featurecla VARCHAR,
    name       NUMERIC,
    map_color6 INT4,
    map_color8 INT4,
    note       VARCHAR,
    zone       NUMERIC,
    utc_format VARCHAR,
    time_zone  VARCHAR,
    iso_8601   VARCHAR,
    places     VARCHAR,
    dst_places VARCHAR,
    tz_name1st VARCHAR,
    tz_namesum INT4,
    cartodb_id INT4,
    created_at VARCHAR,
    updated_at VARCHAR
);
```

创建一张表 `timezones` 来存储我们最终需要的数据

```sql
CREATE TABLE timezones
(
    id            SERIAL,   -- ID
    geom          GEOMETRY, -- 区域
    zone          NUMERIC,  -- 时区
    min_longitude NUMERIC,  -- 最小经度
    max_longitude NUMERIC,  -- 最大经度
    places        VARCHAR,  -- 范围
    PRIMARY KEY (id)
);
```

从 `ne_10m_time_zones` 导入需要的数据

```sql
INSERT INTO timezones(geom, zone, places)
SELECT the_geom::GEOMETRY, zone, places
FROM ne_10m_time_zones
ORDER BY zone;
```

构建区域的最小经度和最大经度

```sql
UPDATE timezones
SET min_longitude = ST_XMin(geom),
    max_longitude = ST_XMax(geom);
```

改进后的根据经纬度计算时区的 PostgreSQL 函数为

```sql
CREATE OR REPLACE FUNCTION calculate_timezone(longitude NUMERIC, latitude NUMERIC) RETURNS NUMERIC AS $$
DECLARE
    timezone NUMERIC;
BEGIN
    -- 从 timezones 表查询时区
    SELECT zone INTO timezone
    FROM timezones
    WHERE ST_Intersects(ST_Point(longitude, latitude, 4326), geom);

    -- 如果未查询到，则使用通用方法计算
    IF timezone IS NULL THEN
        timezone = FLOOR((longitude + 7.5) / 15);
    END IF;

    RETURN timezone;
END;
$$ LANGUAGE plpgsql;
```

在面对有大量数据的表时调用 `calculate_timezone` 函数耗时比较长，比如执行 `UPDATE t SET timezone = calculate_timezone(longitude, latitude);` 语句。上面的函数每次调用时都会遍历 `timezones` 表，使用 `ST_Intersects` 函数计算经纬度是否与在某个区域内，我们期望减少 `ST_Intersects` 函数计算的函数来优化它。实践发现通过在查询时增加 `AND min_longitude <= longitue AND max_longitude >= longitude` 并无效果，因为始终都会访问磁盘遍历 `timezones` 表。另一种方式是把所有的数据放入内存中，通过 `min_longitude` 和 `max_longitude` 过滤出少量的区域再执行 `ST_Intersects` 函数计算。

```sql
CREATE OR REPLACE FUNCTION calculate_timezone(longitude NUMERIC, latitude NUMERIC) RETURNS NUMERIC AS $$
DECLARE
    -- SELECT array_agg('''' || geom || '''::GEOMETRY') FROM (SELECT ST_AsHEXEWKB(ST_SetSRID(geom, 4326)) as geom FROM timezones ORDER BY min_longitude, zone) t;
    geoms GEOMETRY[] = ARRAY[......];
    -- SELECT array_agg(min_longitude) FROM (SELECT min_longitude FROM timezones ORDER BY min_longitude, zone) t;
    min_longitudes NUMERIC[] = ARRAY[......];
    -- SELECT array_agg(max_longitude) FROM (SELECT max_longitude FROM timezones ORDER BY min_longitude, zone) t;
    max_longitudes NUMERIC[] = ARRAY[......];
    -- SELECT array_agg(zone) FROM (SELECT zone FROM timezones ORDER BY min_longitude, zone) t;
    timezones NUMERIC[] = ARRAY[......];
    index INT4;
    candidate_indexs INT4[] = '{}';
    matched_index INT4;
    coordinate GEOMETRY;
    matched BOOL = false;
    timezone NUMERIC;
BEGIN
    -- 构建候选时区对应的下标数组
    FOR index in 1 .. array_length(min_longitudes, 1)
    LOOP
        IF min_longitudes[index] <= longitude AND longitude <= max_longitudes[index] THEN
            candidate_indexs = candidate_indexs || index;
        END IF;
    END LOOP;

    -- 构建坐标点
    coordinate = ST_Point(longitude, latitude, 4326);

    -- 计算坐标点与区域相交时的时区下标
    FOR index in 1 .. array_length(candidate_indexs, 1)
    LOOP
        matched_index = candidate_indexs[index];
        IF ST_Intersects(coordinate, geoms[matched_index]) THEN
            matched = true;
            EXIT;
        END IF;
    END LOOP;

    IF matched THEN
        timezone = FLOOR(timezones[matched_index]);
    ELSE
        -- 没有匹配到时使用通用方法计算
        timezone = FLOOR((longitude + 7.5) / 15);
    END IF;

    RETURN timezone;
END
$$ LANGUAGE plpgsql;
```

受限于篇幅 `geoms`、`min_longitudes`、`max_longitudes` 和 `timezones` 四个变量的内容省略了，它们的内容可以由变量上方的注释的 SQL 语句产生，注意四条 SQL 语句的排序顺序要一致。因为 PostgreSQL 没有 `Map`（映射）类型，我们用多个数组配合数组下标模拟了这种类型，这种处理方法在其他编程语言中也比较常见。

## 参考资料

1. [高中地理：地方时的计算](https://zhuanlan.zhihu.com/p/73670736)
2. [地理 | 区时计算的基本方法](https://zhuanlan.zhihu.com/p/543805210)
3. [高中地理中的时间计算与日期界线](https://zhuanlan.zhihu.com/p/92740299)
4. [时区是怎么划分的？世界各时区的时间如何统一表达？GMT、UTC、UNIX有什么区别？](https://blog.csdn.net/zgdwxp/article/details/102728563)
5. [根据经纬度计算时区](https://blog.csdn.net/fct2001140269/article/details/86513925)
6. [世界时区](https://www.1blueplanet.com/world_time_zones/cn/)
7. [Calculate local time with UTC and location](https://racum.blog/articles/local-time/)
8. [时区 24 个，既有 UTC-12 为何又有 UTC+14？](https://www.zhihu.com/question/321528550)
9. [Natural Earth » Downloads - Free vector and raster map data at 1:10m, 1:50m, and 1:110m scales](https://www.naturalearthdata.com/downloads/)
