---
title: MySQL LBS 实现查找附近的人
date: 2019-08-29 21:03:05
tags: ["MySQL", "坐标排序"]
categories: 服务端开发
---

现在应用中类似 "优先按距离排序" 的功能已经很常见了，那么这些功能如何简单快速的去实现呢？本文将提供一个在数据量不是特别大的时候的解决方案，实现起来比较简单。先将用来演示的表结构和数据初始化一下

<!-- more -->

```SQL
-- 创建商户表
CREATE TABLE merchant (
	id int AUTO_INCREMENT COMMENT '主键',
	merchant_name varchar(64) NOT NULL COMMENT '商户名称',
	merchant_address varchar(64) NOT NULL COMMENT '商户地址',
	longitude double(15, 12) NOT NULL DEFAULT 0.0 COMMENT '经度',
	latitude double(15, 12) NOT NULL DEFAULT 0.0 COMMENT '纬度',
	CONSTRAINT merchant_pk PRIMARY KEY (id)
) COMMENT '商户表';
-- 插入一些测试数据
INSERT INTO merchant VALUES (1, '海底捞火锅', '浦建路118号巴黎春天百货F5', 121.52088, 31.20852);
INSERT INTO merchant VALUES (2, '河马鲜生', '长宁路88号KiNG88广场B1层', 121.42741, 31.22761);
INSERT INTO merchant VALUES (3, '望湘园', '川沙路5398号百联川沙购物中心F3层', 121.69961, 31.18577);
INSERT INTO merchant VALUES (4, '小杨生煎', '川沙路4825号浦东商场B1层', 121.69775, 31.19509);
INSERT INTO merchant VALUES (5, '肯德基', '张江路625号1层B座', 121.61561, 31.20514);
```

然后套用 [Haversine 公式](https://en.wikipedia.org/wiki/Haversine_formula) 直接用 MySQL 的函数去计算距离再排序。例如：假设我当前位置是 **[31.202635, 121.654555]** 查询周边的商家，并且按距离排序。

```SQL
SELECT *, ACOS(
            COS(RADIANS(31.202635)) *
            COS(RADIANS(latitude)) *
            COS(RADIANS(longitude) - RADIANS(121.654555)) +
            SIN(RADIANS(31.202635)) *
            SIN(RADIANS(latitude))
          ) * 6378 AS distance
FROM merchant
ORDER BY distance
LIMIT 0, 20;
```

其中 6378 是地球赤道的半径，如果想调整精度，比如精确到米，可以设置为 6378000，下面是排序的结果

| id  | merchant_name | merchant_address                     | longitude | latitude | distance           |
| :-- | :------------ | :----------------------------------- | :-------- | :------- | :----------------- |
| 5   | 肯德基        | 张江路 625 号 1 层 B 座              | 121.61561 | 31.20514 | 3.7185308288925247 |
| 4   | 小杨生煎      | 川沙路 4825 号浦东商场 B1 层         | 121.69775 | 31.19509 | 4.197812806482076  |
| 3   | 望湘园        | 川沙路 5398 号百联川沙购物中心 F3 层 | 121.69961 | 31.18577 | 4.683026249493325  |
| 1   | 海底捞火锅    | 浦建路 118 号巴黎春天百货 F5         | 121.52088 | 31.20852 | 12.744185472645878 |
| 2   | 河马鲜生      | 长宁路 88 号 KiNG88 广场 B1 层       | 121.42741 | 31.22761 | 21.802509499536363 |
