---
author: pg-x
title: "PostgreSQL 生成等高线"
date: 2023-09-10T19:11:01+08:00
tags: [postgis]
ShowToc: false
TocOpen: false
---

在 [PostGIS](https://github.com/postgis/postgis) 扩展的加持下，PostgreSQL 成为了一个强大的时空数据库。本文介绍一个用 PostGIS 生成[等高线](https://en.wikipedia.org/wiki/Contour_line)的应用示例。

### PostGIS 发展历程

2001 年 5 月，[Refractions Research](http://www.refractions.net/) 发布了 PostGIS 0.1。最初的 PostGIS 版本具备对象、索引和一些函数的功能，适合于存储和检索空间数据，但是缺乏分析能力。随着函数数量的增加，对一个组织来规范接口的需求变得逐渐明朗，开放地理空间联盟（Open Geospatial Consortium）应运而生，并提出了 "Simple Features for SQL"（SFSQL）规范。

接下来的几年，PostGIS 的函数数量不断增加，但其功能仍然有限。其中一些有趣的函数（如 ST_Intersects()、ST_Buffer()、ST_Union()）很难编写，从头开始实现它们需要耗费非常多的时间。幸运的是出现了["Geometry Engine, Open Source" (GEOS)](https://github.com/libgeos/geos) 这个项目。GEOS 提供了实现 SFSQL 规范所需的算法。PostGIS 0.8 版本通过链接 GEOS 库完全支持了 SFSQL。

随着 PostGIS 数据量的增长，出现了另一个问题: 用于存储几何数据的表示形式比较低效。对于点和短线等小对象，元数据开销高达300%。为了提高性能，PostGIS 通过缩小元数据头部和所需维度，降低了开销。在 PostGIS 1.0 中，这种更快速、更轻量的表示形式成为默认选项。

另外，PostGIS 还依赖:

- [Proj](https://github.com/OSGeo/PROJ): 是一个用于地理空间坐标转换的库。它提供各种投影方法和坐标系统的定义，用于在不同的地理空间参考框架之间进行转换。PostGIS 使用 Proj 库来支持空间坐标的转换和投影。
- [GDAL](https://github.com/OSGeo/gdal): Geospatial Data Abstraction Library 是一个用于读取和写入地理空间数据格式的库，包括栅格数据和矢量数据。PostGIS 使用 GDAL 库来支持与各种地理空间数据格式的交互，如 GeoTIFF、Shapefile 等。
- [JSON-C](https://github.com/json-c/json-c): 是一个用于处理 JSON 数据的 C 语言库。它在 PostGIS 中用于处理和解析 GeoJSON 空间数据。
- ...

### 准备数据

在对 PostGIS 简单了解之后，我们准备下用于生成等高线的数据，理想情况下如果有一个区域的坐标海拔数据当然再好不过了，但是我没有找到这样的数据集。不过幸运的是，我找到了一个美国 1987~2022 年 National Lightning Detection Network 公布的闪电数据信息，给的是 0.1 度（大概10km）见方的区域内每天地闪（Cloud to Ground, 简单理解就是劈到地面的闪电才会计数）。

```shell
# 这里只下载了一年的数据
wget https://www1.ncdc.noaa.gov/pub/data/swdi/database-csv/v2/nldn-tiles-2022.csv.gz
# 解压
gunzip nldn-tiles-2022.csv.gz
# 删除文件头，即以 # 开头的行
sed -i '' '/^#/d' nldn-tiles-2022.csv
```

### 建表及导入数据

首先在 PostgreSQL 实例中实验的数据库中安装 postgis 和 postgis_raster 两个插件，然后创建对应的表结构并导入数据:

```SQL
-- Enable PostGIS and PostGIS Raster
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_raster;

-- nldn dataset fields: ZDAY,CENTERLON,CENTERLAT,TOTAL_COUNT
CREATE table nldn_y2022(zday date, lon numeric, lat numeric, strike_cnt int);

-- copy from psql client file system
\copy nldn_y2022 FROM 'nldn-tiles-2022.csv' delimiter ',' csv;

-- 增加一个 geometry 列并更新该列数据
ALTER TABLE nldn_y2022 ADD COLUMN geom geometry(Point);
UPDATE nldn_y2022 SET geom = ST_MakePoint(lon, lat);
```

然后将这一年的数据进行聚合，因为一个坐标点会有多条数据（因为每天都可能会有数据），聚合后的数据放在 nldn_y2022_grid:

```SQL
CREATE table nldn_y2022_grid(geom geometry, strike_cnt)
INSERT INTO nldn_y2022_grid SELECT geom, sum(strike_cnt) FROM nldn_y2022 group by geom;
```

nldn_y2022_grid 的数据可以在 QGIS 软件中查看:

![nldn y2022 data](/images/nldn_y2022_grid_qgis.png)

### 创建栅格（Raster）

生成等高线需要用到栅格数据，因此我们先将上面的数据生成一条栅格记录:

```SQL
-- 获取需要生成栅格的矩形坐标，即四个边界值
CREATE TABLE nldn_y2022_grid_limits(x_min numeric, x_max numeric, y_min numeric, y_max numeric);
INSERT INTO nldn_y2022_grid_limits 
    SELECT min(ST_XMIN(geom)) as xmin, max(ST_XMax(geom)) as xmax, min(ST_YMIN(geom)) as ymin, max(ST_YMAX(geom)) as ymax
        FROM nldn_y2022_grid;

-- 创建栅格表
CREATE TABLE thunder_raster(rid integer, rast raster);
INSERT INTO thunder_raster 
    SELECT 1,
    ST_MakeEmptyRaster(((x_max - x_min) / 0.1)::integer,    -- 栅格的宽度
                       ((y_max - y_min) / 0.1)::integer,    -- 栅格的高度
                       x_min,                               -- 左下角的 x 坐标
                       y_min,                               -- 左下角的 y 坐标
                       0.1,                                 -- x 方向步长 0.1 度
                       0.1,                                 -- y 方向步长 0.1 度
                       0, 0, 4326)
    FROM nldn_y2022_grid_limits;

-- 增加一个 band，用于将 nldn_y2022_grid 中的数据存入其中
UPDATE thunder_raster SET rast = ST_AddBand(rast,'16BUI'::text,0) WHERE rid = 1;

-- 将 nldn_y2022_grid 中的数据更新到 rid = 1 的栅格中的 band 1
UPDATE thunder_raster
SET rast = ST_SetValues(rast,
                        1,
                        (SELECT ARRAY(SELECT (ST_SetSRID(e.geom, 4326), e.strike_cnt)::geomval from nldn_y2022_grid e)))
WHERE rid = 1;
```

如上得到的栅格数据在 QGIS 中的显示如下:

![thunder raster data](/images/thunder_raster_qgis.png)

由于栅格中的数据分布的非常广，0~32237 存储在了一维的 band 中，因此很多数据都显示为了黑色，这张图看起来没什么用处（主要也跟数据集有关，如果是海拔数据可能会好很多）。

### 由 raster 生成等高线

PostGIS 3.2 版本发布了一个 [ST_Contour](https://postgis.net/docs/en/RT_ST_Contour.html) 函数，它是对 [GDAL contouring algorithm](https://gdal.org/api/gdal_alg.html#_CPPv421GDALContourGenerateEx15GDALRasterBandHPv12CSLConstList16GDALProgressFuncPv) 的一个封装。

```SQL
CREATE TABLE contours AS 
    SELECT (ST_Contour(rast,
                      1,        -- band 1
                      fixed_levels => ARRAY[10, 20, 50, 100, 500, 1000, 2000, 10000])).*    -- 对高度为数组中的值生成等高线
    FROM thunder_raster WHERE rid = 1;
```

在 QGIS 中展示的效果如下:

![nldn 2022 contours](/images/nldn_2022_contours.png)

虽然显示出了等高线，但由于数据集是闪电频次，因此没有海拔数据生成的等高线好看。也许对数据进行一下 formalization 后会呈现更好的效果，不过这已经不在本文讨论的范畴了😛。

### 小结

温度、海拔、交通热点等应用都可能会用到等高线来将数据可视化以辅助决策，在生成等高线数据的也学习并调研了很多函数，比如有脏数据的时候（比如没数据的点）可能会用到 [IDW](https://pro.arcgis.com/zh-cn/pro-app/latest/tool-reference/spatial-analyst/idw.htm) 进行插值，这些过程我都记录在了 [lightning_strikes_demo.sh](https://gist.github.com/zhjwpku/dff2f51692d9414b0ddc8193cbd8361c)。

### References:

- [Introduction to PostGIS](https://postgis.net/workshops/postgis-intro/)
- [Building Rasters in PostGIS](https://www.endpointdev.com/blog/2018/09/postgis-raster-generation/)
- [Waiting for PostGIS 3.2: ST_Contour and ST_SetZ](https://www.crunchydata.com/blog/waiting-for-postgis-3.2-st_contour-and-st_setz)
- [TIGER Data Products Guide](https://www.census.gov/programs-surveys/geography/guidance/tiger-data-products-guide.html)
