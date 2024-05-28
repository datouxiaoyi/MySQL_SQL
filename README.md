# 一、嵌套查询优化

当SLQ语句存在嵌套查询时，MySLQ会生成临时表来存储子查询的结果数据，外层查询会从临时表中读取数据，待整个查询完毕后，会删除临时表，在这个过程中是非常耗时的。

方案：使用JOIN语句进行联表查询来代替，提升查询性能。

例如这里查询`t_goods`数据表中`t_category`字段不在`t_goods_category`数据表中的数据。

使用嵌套查询

```sql
mysql> explain select * from t_goods where t_category not in (select t_category from t_goods_category)\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: t_goods
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 35
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 2
  select_type: SUBQUERY
        table: t_goods_category
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 15
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.00 sec)
```

使用`join`联表查询

```sql
mysql> explain select * from t_goods as goods left join t_goods_category as category on goods.t_category_id = category.id where category.t_category is null \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: goods
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 35
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: category
   partitions: NULL
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 15
     filtered: 10.00
        Extra: Using where; Using join buffer (hash join)
2 rows in set, 1 warning (0.00 sec)
```

这里由于数据量的原因，效果可能不是这么明显，但是当查询量非常大时，这种查询效率的效果会更加明显。

# 二、OR条件语句优化

在多个条件使用OR关键字进行连接时，只要OR连接的条件中有一个查询条件没有使用索引，那么MySQL就不会使用索引，而是对数据表进行全表扫描。也就是说，在使用OR连接多个查询条件时，必须每个查询条件都是索引字段，MySQL才会使用索引查询数据。

```sql
mysql> show create table t_goods \G;
*************************** 1. row ***************************
       Table: t_goods
Create Table: CREATE TABLE `t_goods` (
  `id` int NOT NULL AUTO_INCREMENT,
  `t_category_id` int DEFAULT NULL,
  `t_category` varchar(30) DEFAULT NULL,
  `t_name` varchar(50) DEFAULT NULL,
  `t_price` decimal(10,2) DEFAULT NULL,
  `t_stock` int DEFAULT NULL,
  `t_upper_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `index_category_name` (`t_category_id`,`t_name`),
  KEY `category_part` (`t_category`(10)),
  CONSTRAINT `foreign_category` FOREIGN KEY (`t_category_id`) REFERENCES `t_goods_category` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=36 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

例如我有一个由`t_category_id`字段和`t_name`构成联合索引`index_category_name`

如果这里我使用And语句条件来进行查询，那么这里会使用联合索引`index_category_name`。

但是我如果使用OR语句，那么将不会使用索引查询，因为这两个字段无法单独构成索引。

```sql
mysql> explain select * from t_goods where t_name = '耳机' or t_category_id = 1 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: ALL
possible_keys: index_category_name
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 35
     filtered: 12.57
        Extra: Using where
1 row in set, 1 warning (0.01 sec)
```

```sql
mysql> explain select * from t_goods where t_name = '耳机' and t_category_id = 1 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: ref
possible_keys: index_category_name
          key: index_category_name
      key_len: 208
          ref: const,const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

# 三、ORDER BY语句优化

在查询语句中使用`ORDER BY`排序时，如果排序的字段不是索引字段，那么`MySQL`会对结果进行`filesort`文件排序，`filesort`是一种在磁盘上执行的排序算法。如果排序的字段是索引字段，那么会采用索引排序`Index`，会避免额外的操作，提高查询的性能。

```sql
mysql> explain select id,t_stock from t_goods order by t_stock \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: index
possible_keys: NULL
          key: stock_index
      key_len: 5
          ref: NULL
         rows: 35
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
```

这里将商品以价格进行排序，可以看到这里的排序算法为文件排序`Using filesort`

现在我们在这个字段上添加索引

```sql
mysql> alter table t_goods add index stock_index(t_stock);
Query OK, 0 rows affected (0.06 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

现在添加索引后，再次执行查询语句，将商品名以价格进行排序，也就是以字段`t_stock`进行排序

```sql
mysql> explain select id,t_stock from t_goods order by t_stock \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: index
possible_keys: NULL
          key: stock_index
      key_len: 5
          ref: NULL
         rows: 35
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

可以看到这里使用了索引排序。如果查询的字段不在索引字段之内，那么在排序之后依然要进行筛选，所以依然会使用文件排序。（除主键外，因为本身就会使用索引排序）

# 四、GROUP BY语句优化

`GROUP BY`分组，用于数据查询时，对子段进行分组。在`MySQL8.0`之前的版本中，使用`GROUP BY`进行分组查询时，默认会根据分组的字段进行排序，如果该字段上不存在索引，会进行文件排序，比较耗费性能。解决方案：可以指定查询语句通过`ORDER BY NULL`语句来禁止文件排序，减少`ORDER BY`子句带来的性能消耗。

但是在8.0之后的版本中，没有这个默认条件。这里用的是`MySQL8.0`

```sql
mysql> explain select t_category,count(*) from t_goods group by t_category \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 35
     filtered: 100.00
        Extra: Using temporary
1 row in set, 1 warning (0.00 sec)
```

可以看到这里没有使用文件排序，这样避免了filesort排序，会显著提升性能。

# 五、分页查询的优化

`LIMIT`分页查询，指定查询范围。在使用分页查询时，软如果单纯的使用`LIMIT m,n`语句，`MySQL`默认需要排序出数据表中的前`m+n`条数据，然后将前m条数据舍弃，返回`m+1`到`m+n`条数据记录，这样会非常消耗性能。

## 1、回表查询优化

例如，按照`t_upper_time`排序分页查询`t_goods`数据表中的数据

```sql
mysql> explain select id,t_price from t_goods order by t_upper_time limit 1,10 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 35
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
```

直接使用`LIMIT`语句进行分页时，`MySQL`会进行全表扫描并对查询的结果数据使用`filesort`方式进行排序。

现在使用索引分页并回表查询数据

```sql
mysql> explain select g1.id,g1.t_price from t_goods as g1 inner join (select id from t_goods order by t_upper_time limit 1,10) as g2 on g1.id=g2.id \G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 11
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: g1
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: g2.id
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: t_goods
   partitions: NULL
         type: index
possible_keys: NULL
          key: t_upper_time_index
      key_len: 6
          ref: NULL
         rows: 11
     filtered: 100.00
        Extra: Using index
3 rows in set, 1 warning (0.00 sec)
```

这里使用查询嵌套，将子查询的结果集（派生表）作为中间表与`g1`也就是`goods`表进行`join`连接，形成最终表。这里的三条记录分别代表步骤。

第一条记录（最终的子查询），展示了从执行子查询到最终进行连接和返回结果的整个过程

第二条记录（主查询的一部分），也就是`g1`表与派生表之间的嵌套查询，通过`id`进行条件筛选。

第三条记录（子查询），也就是`g2`派生表，在这个查询中使用的索引排序并且使用逐渐筛选。

这样会大大提高查询的效率。不需要扫瞄全表。

## 2、记录数据标识优化分页

简单来说就是通过id值来查询数据的索引位置，当数据量特别大时，可以记录分页数据的最后一条数据的Iid值，当需要查询下一页时，只需要查询id的值大于记录的id值的n条数据即可。

例如，这里的每页查询10条记录，现在我需要查询第二页，那么我只需要将条件设置为id>20即可，向后取10个数据，再次记录id值，以此类推。

```sql
mysql> explain select id,t_price from t_goods force index (t_upper_time_index) where id>20 order by t_upper_time limit 10 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_goods
   partitions: NULL
         type: index
possible_keys: NULL
          key: t_upper_time_index
      key_len: 6
          ref: NULL
         rows: 10
     filtered: 33.33
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

可以看到这样页可以大大提高查询的性能，但是这种优化方式有限定条件，必须保证排序字段的索引不会重复，如果出现了重复，则可能会丢失部分数据。

# 六、插入数据优化

向数据表中插入数据时，如果数据表中存在索引、唯一性校验时，会影响插入数据的效率。

## 1、MyISAM数据表插入数据优化

### 1.1、禁用索引

插入大量数据时，数据表中存在索引会降低数据插入的性能，可以在插入数据前禁用索引，待数据插入完毕再开启索引。

```sql
alter table t_table_name disable keys;    #禁用索引
alter table t_table_name enable keys;    #开启索引
```

### 1.2、禁用唯一性检查

向数据表中插入数据时，对数据进行唯一性检查也会降低数据的插入性能，和索引一样，可以在插入数据前禁用唯一性检查，插入完毕后再开启。

```
set unique_checks = 0 ；    #禁用唯一性检查
set unique_checks = 1 ；    #开启唯一性检查
```

### 1.3、禁用外键检查

```
set foreign_key_checks = 0 ;     #禁用外键检查
set foreign_key_checks = 1 ;     #开启外键检查
```

### 1.4、批量插入数据

向数据表中韩插入多条数据时，使用insert语句一次插入多条数据比每次插入一条数据性能要高很多。

```sql
INSERT INTO t_goods_category (id, t_category, t_remark) VALUES
(6, '食品', '各种食品和饮料'),
(7, '玩具', '儿童和成人的玩具'),
(8, '家具', '家庭和办公室家具'),
(9, '办公用品', '办公设备和文具'),
(10, '宠物用品', '宠物的食品和用品');
```

### 1.5、批量导入数据

使用LOAD DATA INFILE 语句向数据表中导入数据比使用INSERT语句的性能高，适用于需要批量导入的数据量很大时。

```sql
LOAD DATA INFILE 'data_file_path' into table table_name;
```

## 2、InnoDB数据表插入数据优化

这里，MyISAM中适用的优化，在这里也适用。但是需要注意的是，在批量导入数据时，被导入的文件中的数据记录最好是按主键顺序排列，这样可以提高导入数据的效率。

除此之外，InnoDB是支持事务的，可以在插入数据之前禁用MySQL自动提交事务，待插入数据完成后，在开启事务的自动提交，这样可以提高数据的插入性能。

```sql
set autocommit = 0 ;    #禁用事务自动提交
set autocommit = 1 ;    #开启事务自动提交
```

# 七、删除数据的优化

如果数据表是分区表，删除数据表中的数据时，如果待删除的数据正好是数据表中某个分区的所有数据，这样可以直接删除分区，比DELETE语句删除数据性能要高很多。

```sql
alter table table_name drop partition partition_name;    #删除分区
```

注意，通过删除分区的方式删除数据只适用于range、list、range columns、list columns分区。