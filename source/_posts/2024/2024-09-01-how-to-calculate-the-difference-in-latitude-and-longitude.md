---
title: 如何计算经纬度的差？
date: 2024-09-01 16:54:24
updated: 2024-09-01 16:54:24
tags: [GIS]
---

为了方便定位地球上的点，人为定义了经线和纬线，它们组成了一个经纬网。经度的范围为 [-180°, +180°]，本初子午线的位置为 0°，向东走为东经，经度范围为 [0°, +180°]，向西走为西经，经度范围为 [-180°, 0°]，-180° 和 180° 对应同一根经线，也叫国际日期变更线。纬度的范围为 [-90°, +90°]，赤道的位置为 0°，向北走为北纬，纬度范围为 [0°, +90°]，向南走为南纬，纬度范围为 [-90°, 0°]，-90° 和 90° 的位置对应南极点和北极点。

{% asset_img longitude-and-latitude.jpg %}

我们的问题是对于给定两个经度或纬度，如何计算它们之间的差值？

<!-- more -->

我们来复习一下如何计算数轴上两个点的距离

{% asset_img point-distance.jpg %}

由于纬度和数轴类似，因此两个纬度之间的差可以类比数轴上两个点的距离进行计算

```java
public static double calculateLatitudeDifference(double lat1, double lat2) {
    return Math.abs(lat1 - lat2);
}
```

对于经度我们可以进行类似的计算

```java
public static double calculateLongitudeDifference(double lon1, double lon2) {
    return Math.abs(lon1 - lon2);
}
```

在上面的计算方式下 +170° 和 -170° 之间差为 340°，我们把世界地图打开，+170° 和 -170° 之间是很近的，它们是国际日期变更线两侧 10° 的经线，它们之间的差为 20° 看起来才合理。

{% asset_img world-map.jpg %}

其实上面的计算方式不能算是错误的，对于纬线来说它们只有在赤道的 0° 纬线是重叠的，但是对于经线来说它在本初子午线的 0° 经线和国际日期变更线的 +180°（-180°）两个位置重叠了。按照上面的计算方式计算的是两个经度最远的距离，对于经度来说我们期望计算它们最近的距离

```java
public static double calculateLongitudeDifference(double lon1, double lon2) {
    double difference = Math.abs(lon1 - lon2);
    if (difference > 180) {
        difference = 360 - difference;
    }
    return difference;
}
```

完～
