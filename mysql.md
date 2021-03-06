# mysql知识整理
- 基础知识    
  - 普通查询语句
  - join
  - 子查询
  - union
  - union all
  - [触发器](#触发器)
  - 存储过程
- 数据库优化
  - 应用系统sql优化
    - 分表策略
    - 分库策略
    - 索引
    - 普通索引，联合索引
    - 如何选择字段创建索引
    - 索引失效原因 
    - 查询缓存
  - mysql服务器优化
    - 开启慢查询定位问题
    - mysql连接数
  - 操作系统与硬件优化
  - 系统架构整体优化
    - 负载均衡
    - 缓存
    - 分布式优化
- 数据管理与维护
  - 数据备份
  - mysqldump备份数据库
  - mysldump备份数据表
  - mysqldump备份数据表结构
  - 数据恢复
  - mysql日志
  - mysql监控
    - mysql常用工具



## 触发器
触发器的定义：触发器（TRIGGER）是MySQL的数据库对象之一，从5.0.2版本开始支持。该对象与编程语言中的函数非常类似，都需要声明、执行等。但是触发器的执行不是由程序调用，也不是由手工启动，而是由事件来触发、激活从而实现执行。

#### 触发器的基本语法
```
CREATE TRIGGER trigger_name
trigger_time
trigger_event ON tbl_name
FOR EACH ROW
begin
……
end
```

这里触发器的有两种:before,after
触发的事件类型为：update，insert，delete
#### 创建两张表并插入一些数据演示下触发器的使用
```
CREATE TABLE goods (
        id INT(11) UNSIGNED NOT NULL auto_increment,
        name VARCHAR(40) not null COMMENT '商品名称',
        stock SMALLINT(11) UNSIGNED NOT NULL COMMENT '商品库存',
        PRIMARY KEY(`id`)
        )ENGINE=INNODB DEFAULT CHARSET=utf8 COMMENT '商品表' ;
```

```
INSERT INTO goods (`name`,`stock`) values ('iphonex',50),('小米2',30),('联想手机',40);
```

```
CREATE TABLE goods_order(
        oid int(11) UNSIGNED NOT NULL auto_increment COMMENT '订单id,自增id',
        gid INT(11) UNSIGNED NOT NULL COMMENT 'goods表的商品id',
        nums SMALLINT(11) UNSIGNED NOT NULL COMMENT '订单购买数量',
        PRIMARY KEY(`oid`)
        ) ENGINE=INNODB DEFAULT CHARSET=utf8 COMMENT '商品订单';
```

#### 创建触发器当有新订单的时候减少goods表的库存数量
```
CREATE TRIGGER t1
AFTER
INSERT 
ON goods_order
FOR EACH ROW
BEGIN
UPDATE goods SET stock = stock-new.nums WHERE id=new.gid;
END
```

#### 运行创建触发器语句后显示触发器
```
MySQL [test]> show triggers\G;
*************************** 1. row ***************************
Trigger: t1
Event: INSERT
Table: goods_order
Statement: BEGIN
UPDATE goods SET stock = stock-new.nums WHERE id=new.gid;
END
Timing: AFTER
Created: NULL
sql_mode: NO_ENGINE_SUBSTITUTION
Definer: root@%
character_set_client: utf8
collation_connection: utf8_general_ci
    Database Collation: utf8_unicode_ci
1 row in set (0.00 sec)
```

#### 查询当前数据库商品表和订单表的数据
```
MySQL [test]> select * from goods;
+----+--------------+-------+
| id | name         | stock |
+----+--------------+-------+
|  1 | iphonex      |    50 |
|  2 | 小米2        |    30 |
|  3 | 联想手机     |    40 |
+----+--------------+-------+
3 rows in set (0.00 sec)

    MySQL [test]> select * from goods_order;
Empty set (0.00 sec)
```

#### 向订单表中插入数据查看商品表库存是否自动减掉了
```
MySQL [test]> insert into goods_order (`gid`,`nums`) values (1,2);
Query OK, 1 row affected (0.00 sec)

MySQL [test]> select * from goods;
+----+--------------+-------+
| id | name         | stock |
+----+--------------+-------+
|  1 | iphonex      |    48 |
|  2 | 小米2        |    30 |
|  3 | 联想手机     |    40 |

MySQL [test]> select * from goods_order;
+-----+-----+------+
| oid | gid | nums |
+-----+-----+------+
|   1 |   1 |    2 |
+-----+-----+------+
```
经过对照发现当添加一条商品，购买数量为2时商品表的库存数由50变为了48说明触发器运行成果，和我们所猜想的一致。

#### 想订单表中插入超过库存总量的数据时测试效果
```
MySQL [test]> insert into goods_order (`gid`,`nums`) values (2,32);
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(`test`.`goods`.`stock` - NEW.nums)'

插入超过库存数量的商品时sql报错，因为我们设置的字段是无符号的，如果当前字段是有符号此时，字段会更新为-2，这肯定是不符合我们要求的，此时需要修改t1触发器来，当超过库存总数量的时候我们减掉最大库存即可。
```
#### 修改t1触发器
```
DROP TRIGGER t1;
CREATE TRIGGER t1
BEFORE
INSERT 
ON goods_order
FOR EACH ROW
BEGIN
DECLARE rnum int;
##查询插入之前商品的库存
SELECT stock into rnum from goods where id=new.gid;
##如果即购买订单的商品数量大于总库存，则设置为购买的数量为当前的商品库存数量
if new.nums>rnum THEN
    SET new.nums = rnum;
END IF;
UPDATE goods SET stock = stock-new.nums WHERE id=new.gid;
END
```

#### 插入订单查看效果
```
MySQL [test]> select * from goods;
+----+--------------+-------+
| id | name         | stock |
+----+--------------+-------+
|  1 | iphonex      |    48 |
|  2 | 小米2        |    30 |
|  3 | 联想手机     |    40 |
+----+--------------+-------+
3 rows in set (0.00 sec)

MySQL [test]> select * from goods_order;
+-----+-----+------+
| oid | gid | nums |
+-----+-----+------+
|   1 |   1 |    2 |
+-----+-----+------+
1 row in set (0.00 sec)


MySQL [test]> insert into goods_order (`gid`,`nums`) values (2,32);
Query OK, 1 row affected (0.00 sec)

MySQL [test]> select * from goods;
+----+--------------+-------+
| id | name         | stock |
+----+--------------+-------+
|  1 | iphonex      |    48 |
|  2 | 小米2        |     0 |
|  3 | 联想手机     |    40 |
+----+--------------+-------+
3 rows in set (0.00 sec)

MySQL [test]> select * from goods_order;
+-----+-----+------+
| oid | gid | nums |
+-----+-----+------+
|   1 |   1 |    2 |
|   3 |   2 |   30 |
+-----+-----+------+
2 rows in set (0.00 sec

```
从上门可以看出，当我们订单中购买数量大于商品库存总数的时候，商品库存只会扣除最大库存数

#### 创建触发t2，实现当删除订单表时恢复商品库存数量
```
CREATE TRIGGER t2
AFTER
DELETE
ON goods_order
FOR EACH ROW
BEGIN
UPDATE goods SET stock=stock+old.nums where id=old.gid; 
END
```

查询当前所有触发器
```
MySQL [test]> show triggers\G;
*************************** 1. row ***************************
             Trigger: t1
               Event: INSERT
               Table: goods_order
           Statement: BEGIN
        DECLARE rnum int;
 ##查询插入之前商品的库存
        SELECT stock into rnum from goods where id=new.gid;
 ##如果即购买订单的商品数量大于总库存，则设置为购买的数量为当前的商品库存数量
        if new.nums>rnum THEN
                SET new.nums = rnum;
        END IF;
        UPDATE goods SET stock = stock-new.nums WHERE id=new.gid;
END
              Timing: BEFORE
             Created: NULL
            sql_mode: NO_ENGINE_SUBSTITUTION
             Definer: root@%
character_set_client: utf8
collation_connection: utf8_general_ci
  Database Collation: utf8_unicode_ci
*************************** 2. row ***************************
             Trigger: t2
               Event: DELETE
               Table: goods_order
           Statement: BEGIN
UPDATE goods SET stock=stock+old.nums where id=old.gid; 
END
              Timing: AFTER
             Created: NULL
            sql_mode: NO_ENGINE_SUBSTITUTION
             Definer: root@%
character_set_client: utf8
collation_connection: utf8_general_ci
  Database Collation: utf8_unicode_ci
2 rows in set (0.00 sec)
```

删除订单表id=1的数据
```
MySQL [test]> delete from goods_order where id=1;
ERROR 1054 (42S22): Unknown column 'id' in 'where clause'
MySQL [test]> delete from goods_order where oid=1;
Query OK, 1 row affected (0.00 sec)

MySQL [test]> select * from goods;
+----+--------------+-------+
| id | name         | stock |
+----+--------------+-------+
|  1 | iphonex      |    50 |
|  2 | 小米2        |     0 |
|  3 | 联想手机     |    40 |
+----+--------------+-------+
3 rows in set (0.00 sec)

MySQL [test]> select * from goods_order;
+-----+-----+------+
| oid | gid | nums |
+-----+-----+------+
|   3 |   2 |   30 |
+-----+-----+------+
1 row in set (0.00 sec)
```
由上门可以看出，删除id=1的数据后，订单表id=1的商品库存数由之前48增加到了50说明触发器成功执行

#### 最后解释下FOR EACH ROW
for each row表示的是执行的触发起的动作影响了多少行数据就执行多少行的数据
