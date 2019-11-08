title: Mysql-时间计算
author: whsoftinc
tags:
  - Mysql
  - 数据库
categories:
  - 数据库
  - Mysql
toc: true
thumbnail: /gallery/thumbnails/mysql1.png
date: 2019-08-29 21:11:00
---
<Excerpt in index>
<!-- more -->

# Mysql日期计算、返回月、周、日
## 一、引言
当我们使用mysql数据库查询时，有时候业务会根据时间的年月日进行条件筛选。  
## 二、 查询2个日期之间相差的天数
```mysql
    SELECT DATEDIFF('2019-08-28', '2019-08-20'); -- 8
    SELECT DATEDIFF(DATE_FORMAT(now(),'%Y-%m-%d'), '2019-08-20'); -- 查询当前时间与传入的时间比对
    SELECT TIMESTAMPDIFF(DAY, '2019-09-05 12:12:13', '2019-09-06 12:12:12') -- 0 计算精确到时分秒
```

## 三、 查询2个日期之间相差的周数
```mysql
    SELECT FLOOR(DATEDIFF('2019-08-28', '2019-07-01')/7) -- 8
    SELECT FLOOR(DATEDIFF(DATE_FORMAT(now(),'%Y-%m-%d'), '2019-07-01')/7) -- 8 查询当前时间与传入的时间比对
    SELECT TIMESTAMPDIFF(WEEK, '2019-08-30 12:12:13', '2019-09-06 12:12:12') -- 0 计算精确到时分秒 
```
## 四、查询2个日期之间相差的月数
```mysql
    SELECT PERIOD_DIFF(DATE_FORMAT('2019-08-05','%Y%m'),DATE_FORMAT('2019-05-03','%Y%m')) --3
    
    SELECT PERIOD_DIFF(DATE_FORMAT(now(),'%Y%m'),DATE_FORMAT('2019-05-03','%Y%m')) -- 查询当前时间与传入的时间比对
    SELECT TIMESTAMPDIFF(MONTH, '2019-08-30 12:12:13', '2019-09-30 12:12:12') -- 0 计算精确到时分秒 
```

## 五、查询2个日期之间相差的时间，返回 time 差值。 
```mysql
    SELECT TIMEDIFF('2019-08-08 08:08:08', '2019-08-07 00:00:00'); -- 32:08:08  
    SELECT TIMEDIFF('08:08:08', '00:00:00'); -- 08:08:08 
```
## 六、查询本周、上周、本月、上月的数据 
* 本周
```mysql
    SELECT name,time FROM a WHERE YEARWEEK(DATE_FORMAT(time,'%Y-%m-%d')) = YEARWEEK(now());
```
* 上周
```mysql
    SELECT name,time FROM a WHERE YEARWEEK(DATE_FORMAT(time,'%Y-%m-%d')) = YEARWEEK(now())-1;
```
* 本月
```mysql
    SELECT NAME, time FROM a WHERE DATE_FORMAT(time, '%Y-%m') = DATE_FORMAT(now(), '%Y-%m')
```
* 上月
```mysql
    SELECT NAME, time FROM a WHERE DATE_FORMAT(time, '%Y-%m') = DATE_FORMAT(DATE_SUB(NOW(), INTERVAL 1 MONTH),'%Y-%m')
```