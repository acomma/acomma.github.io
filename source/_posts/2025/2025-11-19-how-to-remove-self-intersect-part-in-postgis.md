---
title: 如何在 PostGIS 中删除 LINESTRING 自相交部分
date: 2025-11-19 20:59:53
updated: 2025-11-19 20:59:53
tags: [PostGIS]
---

假设有如图一所示的线串，它在线段 2 和 6 处相交，如何把相交的部分删除得到如图三所示的简单线串呢？

{% asset_img self-intersect.drawio.svg %}

<!-- more -->

一种处理方式是遍历每一条线段并判断该线段是否与后面的线段相交，如果相交则删除该线段与相交线段之间所有的线段，如图二所示删除线段 2、3、4、5、6 就可以得到结果。这种处理方式在 PostGIS 中实现如下：

```sql
CREATE OR REPLACE FUNCTION ST_RemoveIntersection(line GEOMETRY)
RETURNS GEOMETRY
AS $$
DECLARE
    result GEOMETRY;
BEGIN
    -- 存储线串所有的点和线段
    CREATE TEMPORARY TABLE points
    (
        index INTEGER,    -- 点的索引，也是线段的索引
        point GEOMETRY,   -- 点
        segment GEOMETRY, -- 线段
        PRIMARY KEY (index)
    )
    ON COMMIT DROP;

    -- 创建索引加快长线串的处理速度
    CREATE INDEX points_segment_gist ON points USING GIST(segment);

    -- 将线串拆分为点和线段，最后一个点没有与之匹配的线段
    INSERT INTO points(index, point, segment)
    SELECT t.path[1] AS index, t.geom AS point, ST_MakeLine(t.geom, LEAD(t.geom, 1) OVER()) AS segment
    FROM ST_DumpPoints(line) AS t;

    WITH remove_indexs AS (
        -- 找出相交的线段的索引，也是要删除的点的索引
        SELECT a.index AS start_index, b.index AS end_index
        FROM points AS a
            LEFT JOIN points AS b
                -- 两条线段之间至少隔了一条线段，即相邻线段不算相交
                ON b.index >= a.index + 2
        WHERE a.segment IS NOT NULL
            AND b.segment IS NOT NULL
            -- 判断线段是否相交
            AND ST_Intersects(a.segment, b.segment)
    )
    -- 剩下的点构成新的线串
    SELECT ST_MakeLine(p.point)
    INTO result
    FROM points as p
    WHERE NOT EXISTS(
        -- 删除相交的线段，也是要删除的点
        SELECT 1 FROM remove_indexs AS r WHERE r.start_index < p.index AND p.index <= r.end_index
    );

    RETURN result;
END;
$$
    LANGUAGE plpgsql;
```
