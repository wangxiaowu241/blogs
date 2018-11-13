# 一、关于 IS NULL & IS NOT NULL 索引是否生效

表结构信息如下:

![1529039325365](C:\Users\ADMINI~1\AppData\Local\Temp\1529039325365.png)

```
CREATE TABLE `t_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `password` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `del_flg` int(11) DEFAULT '0',
  PRIMARY KEY (`id`),
  KEY `CREATE_TIME_INDEX` (`create_time`)
) ENGINE=InnoDB AUTO_INCREMENT=251265 DEFAULT CHARSET=utf8;
```

可以看到：id和create_time字段是有索引的。

以下为四条id和create_time分别在 IS NULL & IS NOT NULL时执行计划的结果。

| sql语句                                                    | 索引是否生效 |                                                              |
| ---------------------------------------------------------- | ------------ | ------------------------------------------------------------ |
| EXPLAIN SELECT * FROM t_user where create_time is NULL     | 生效         | ![1529039710781](C:\Users\ADMINI~1\AppData\Local\Temp\1529039710781.png) |
| EXPLAIN SELECT * FROM t_user where create_time is NOT NULL | 生效         | ![1529039738175](C:\Users\ADMINI~1\AppData\Local\Temp\1529039738175.png) |
| EXPLAIN SELECT * FROM t_user where num is NULL             | 生效         | ![1529040322592](C:\Users\ADMINI~1\AppData\Local\Temp\1529040322592.png) |
| EXPLAIN SELECT * FROM t_user where num is NOT NULL         | 生效         | ![1529040359632](C:\Users\ADMINI~1\AppData\Local\Temp\1529040359632.png) |

事实证明：IS NULL  && IS NOT NULL 索引是生效的。那为什么强烈建议不要使用null作为字段的默认值呢？
  
