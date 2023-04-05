## 高性能索引

索引(MySQL中也叫键key)是存储引擎用于快速找到记录的一种数据结构。

索引优化应该是对查询性能优化最有效的手段。

### 语法

MySQL的**key**同时具有constraint和index的意义，也就是既有约束作用，也有索引作用。

- Primary Key：起到主键和索引的作用
- Unique Key：规范数据的唯一性，同时有索引作用
- Foreign Key：外键，规范数据的引用完整性，同时有索引作用

index则是纯粹的index。

直接创建索引：
```
CREATE [UNIQUE | FULLTEXT | SPATIAL] INDEX index_name
    [index_type]
    ON tbl_name (key_part,...)
    [index_option]
    [algorithm_option | lock_option] ...

key_part:
    col_name [(length)] [ASC | DESC]

index_option:
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'

index_type:
    USING {BTREE | HASH}

algorithm_option:
    ALGORITHM [=] {DEFAULT | INPLACE | COPY}

lock_option:
    LOCK [=] {DEFAULT | NONE | SHARED | EXCLUSIVE}
```
创建table的时候指定索引：
```
CREATE TABLE tbl_name (
    col_name column_definition，
    {INDEX|KEY} [index_name] [index_type] (key_part,...)
)

key_part:
    col_name [(length)] [ASC | DESC]
```
创建时指定主键：
```
CREATE TABLE tbl_name (
    col_name column_definition，
    [CONSTRAINT [symbol]] PRIMARY KEY [index_name] [index_type] (key_part,...)
)
```
创建时指定唯一索引：
```
CREATE TABLE tbl_name (
    col_name column_definition，
    [CONSTRAINT [symbol]] UNIQUE [INDEX|KEY] [index_name] [index_type] (key_part,...)
)
```
创建时指定外键：
```
CREATE TABLE tbl_name (
    col_name column_definition，
    [CONSTRAINT [symbol]] FOREIGN KEY [index_name] (col_name,...)
    reference_definition
)
```
已存在表更新添加索引：
```
ALTER TABLE tbl_name
    | ADD {INDEX|KEY} [index_name] [index_type] (key_part,...) [index_option]
    | ADD [CONSTRAINT [symbol]] PRIMARY KEY [index_type] (key_part,...) [index_option] ...
    | ADD [CONSTRAINT [symbol]] UNIQUE [INDEX|KEY] [index_name] [index_type] (key_part,...) [index_option] ...
    | ADD [CONSTRAINT [symbol]] FOREIGN KEY [index_name] (col_name,...) reference_definition
```
撤销索引：
```
ALTER TABLE tbl_name
    | DROP {INDEX|KEY} index_name
    | DROP PRIMARY KEY
    | DROP FOREIGN KEY fk_symbol
```

### 索引基础

存储引擎现在索引中找到对应值，然后根据匹配的索引记录找到对应的数据行。

索引可以包含一个或多个列的值。多个列的顺序也十分重要，MySQL只能高效的使用索引的最左前缀列。

#### 索引类型

索引是在存储引擎层实现的。

**B-Tree**索引

使用B-Tree数据结构来存储数据：

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/database/images/b-tree-index.png)

B-Tree通常意味着所有值都是按照顺序存储的，每一个叶子页到根的距离相同。根节点的槽中存放了指向子节点的指针。叶子节点的指针指向的是被索引的数据，而不是其他的节点页。根节点和叶子节点之间可能有很多层节点页，树的深度和表的大小直接相关。

B-Tree是顺序存储的，所以很适合查找范围数据。

B-Tree适用于全键值、键值范围或键前缀查找：

- 全值匹配：和索引中的所有列进行匹配
- 匹配最左前缀：只匹配索引的第一列或前两列
- 匹配列前缀：匹配开头列的开头部分
- 匹配范围值
- 精确匹配某一列并范围匹配另外一列：第一列全匹配，第二列范围匹配

限制：

- 不是按照索引的最左列开始查找，则无法使用索引
- 不能跳过索引中的列
- 查询中如果有某个列的范围查找，则右边所有列都无法使用索引优化查找

可见，索引的**顺序**非常重要，这些限制都和顺序有关。

**哈希索引**

基于哈希表实现，只有精确匹配索引所有列的查询才会有效。只有Memory引擎显示支持哈希索引。
```
CREATE TABLE tbl_name (
    KEY USING HASH(colname)
);
```
哈希无法用于排序，不支持部分索引，只支持等值比较(`=`,`IN`,`<=>`)。速度非常快。

InnoDB引擎有个特殊功能叫自适应哈希。当某些索引值被使用的非常频繁时，会在内存中基于B-Tree的索引之上再创建一个哈希索引。

我们也可以自己在B-Tree基础上创建一个伪哈希索引。在B-Tree中使用哈希值而不是键本身。比如一个很长的url列，可以新增一个被索引的url_crc列，使用CRC32做哈希。
建表：
```
CREATE TABLE pseudohash ( 
  id int unsigned NOT NULL auto_increment, 
  url varchar(255) NOT NULL, 
  url_crc int unsigned NOT NULL DEFAULT 0, 
  PRIMARY KEY( id) 
);
```
创建触发器，自动生成crc：
```
DELIMITER // 
    CREATE TRIGGER pseudohash_crc_ins BEFORE INSERT 
    ON pseudohash FOR EACH ROW BEGIN SET NEW.url_crc= crc32(NEW.url);
END;
// 
    CREATE TRIGGER pseudohash_crc_upd BEFORE UPDATE  
    ON pseudohash FOR EACH ROW BEGIN SET NEW.url_crc= crc32(NEW.url); 
END; 
// 

DELIMITER ;
```
查询：
```
SELECT id FROM url 
WHERE url="http://www.mysql.com" 
AND url_crc=CRC32("http://www.mysql.com");
```
必须使用常量url，是为了避免哈希冲突。

### 索引优点

B-Tree索引，按照顺序存储数据，所以可以用来做ORDER BY和GROUP BY操作。索引中存储了实际的列值，某些查询只使用索引就能完成全部查询。

优点：

- 大大减少了服务器需要扫描的数据量
- 帮助服务器避免排序和临时表
- 将随机I/O变为顺序I/O

三星系统：将相关的记录放在一起获得一星，索引中的数据顺序和查找中的排列顺序一致则获得两星，索引中的列包含了查询中需要的全部列则获得三星。

对于非常小的表，大部分情况下全表扫描更高效。中大型的表，索引就非常有效。非常大的表，可以分区，如果表的数量特别多，可以建立一个元数据信息表。

### 高性能索引策略

有些是针对特殊案例的优化方法，有些则是针对特定行为的优化。

#### 独立的列

独立的列是指索引列不能是表达式的一部分，也不能是函数的参数，而是独立的。`WHERE actor_id + 1 = 5`是不行的。

#### 前缀索引和索引选择性

索引很长的列时，可以使用模拟哈希索引，也可以进行前缀索引。

前缀索引是索引开始的部分字符，可以大大节约索引空间，但降低了索引的选择性。

选择性是指不重复的索引值和数据表的记录总数的比值。索引的选择性越高则查询效率额越高。唯一索引的选择性是1，这是性能最好的。

对于BLOB、TEXT或者很长的VARCHAR类型的列，必须使用前缀索引。诀窍在于要选择足够长的前缀保证选择性，又不能太长。前缀的基数应该接近于完整列的基数。

计算完整列的选择性：
```
SELECT COUNT(DISTINCT colname)/COUNT(*) FROM table_name;
```
查询前缀的选择性：
```
SELECT COUNT(DISTINCT LEFT(colname, 3))/COUNT(*) FROM table_name;
```
可以从小到大查看前缀的选择性，查清到多少时选择性提升幅度变小。

只看平均选择性也是不够的，如果数据分布不均匀，可能会有陷阱。比如都以`San`开头的城市，可以查看平均性：
```
SELECT COUNT（*) AS cnt, LEFT(city, 4) AS pref 
FROM table_name 
GROUP BY pref
ORDER BY cnt DESC
LIMIT 5
```

创建前缀索引：
```
ALTER TABLE sakila.city_demo ADD KEY (city(7));
```
使用：
```
select * from city_demo where city like 'ShangH%';
```
前缀索引可以使索引更小更快，但无法用来做ORDER BY和GROUP BY，也无法用来做覆盖扫描。

一个很常见的场景是针对很长的十六进制唯一ID使用前缀索引。

有时候后缀索引也很有用，比如电子邮箱。可以将字符串反转后存储。

#### 多列索引

经常见的错误是为每个列单独建立索引或者使用错误的顺序建立多列索引。

新版本MySQL中查询能够同时使用两个单列做引进行扫描，并将结果进行合并。索引合并有时候是一种优化的结果，但实际上更多时候说明了表的索引建的很糟糕。

- 多个索引做相交操作时(多个AND条件)，意味着需要一个包含所有相关列的多列索引。
- 对多个索引做联合操作时(多个OR)，通常需要耗费大量CPU和内存资源在算法的缓存、排序和合并上。
- 优化器不会把这些成本算到查询成本中，使得成本被低估

在EXPLAIN中看到有索引合并，应该好好检查一下表的索引。

#### 合适的索引列顺序

有一个经验法则，在不考虑排序和分组的情况下，将选择性最高的列放在索引最前列。这时候索引的作用只是用于优化WHERE条件的查找，对于在WHERE子句中只是用了索引部分前缀列的查询来说选择性也更高。

有时候还要考虑值得分布。可能需要根据那些运行频率最高的查询来调整索引列的顺序。

#### 聚簇索引

聚簇索引并不是一种单独的索引类型，而是一种数据存储方式。InnoDB的聚簇索引实际上在同一个结构中保存了B-Tree索引和数据行，数据行实际保存在索引的叶子页中。聚簇表示数据行和相邻的键值紧凑的存储在一起。

一个表只能有一个聚簇索引。

聚簇索引中叶子页包含了行的全部数据，节点页只包含了索引列。

InnoDB通过主键聚集数据，如果没有定义主键，会选择一个唯一的非空索引代替，如果没有这样的索引，会隐式的定义一个主键。

优点：

- 相关数据保存在一起
- 数据访问更快，索引与数据在一起
- 覆盖索引的查询可以直接使用页节点的中的主键值。

缺点：

- 提高了I/O密集型应用的性能，如果数据全部放在内存中，就没什么优势了
- 按照主键的顺序插入最快，如果不按照主键顺序，加载完成后最后使用OPTIMIZE TABLE重新组织表
- 更新聚簇索引的代价很高，需要移动整个行
- 主键更新，可能引起页分裂
- 聚簇索引可能会使全表扫描变慢
- 二级索引可能会变大，因为要包含引用行的主键值
- 二级索引需要两次索引查找，先找到二级索引的叶子节点保存的主键值，然后根据这个值去聚簇索引查找相应的行

MyISAM中的主键索引和其它索引结构上没什么不同，只是一个叫Primary的唯一非空索引

对于InnoDB，聚簇索引就是表。没一个叶子节点包含了主键值,事务ID、用于事务和MVCC的回滚指针及所有的剩余列。二级索引的叶子节点存储的不是行指针，而是主键值。

最简单的方法是使用AUTO_INCREMENT自增列，可以保证数据行是顺序写入。最好避免随机的聚簇索引。

#### 覆盖索引

如果一个索引包含或者覆盖所有需要查询的字段的值，称为覆盖索引。查询只需要扫描索引而无须回表，会带来极大好处。

覆盖索引必须存储索引列的值，哈希索引、空间索引、全文索引都不存储索引列的值，只有B-Tree能用来做覆盖索引。

EXPLAIN的Extra显示Using index。

MySQL能在索引中做最左前缀匹配的LIKE操作，但如果是通配符开头的LIKE操作，无法匹配。

InnoDB的二级索引叶子节点包含了主键值，可以利用这些额外的主键列来覆盖查询。

假设索引覆盖了WHERE条件中的字段，但不是整个查询涉及的字段，条件为false，MySQL5.5及更老的版本也会回表获取数据行。解决办法是先查询索引的id，再进行二次查询：
```
SELECT *  FROM products 
    JOIN (  
          SELECT prod_id 
          FROM products 
          WHERE actor='SEAN CARREY' AND title LIKE '%APOLLO%' 
    ) AS t1 
    ON (t1.prod_id= products.prod_id)
```
这叫做延迟关联，使用了部分覆盖索引。

上面提到的很多限制都是由于存储引擎API设计所导致的，目前的API设计不允许MySQL将过滤条件传到存储引擎层。如果MySQL在后续版本能够做到这一点，则可以把查询发送到数据上，而不是像现在这样只能把数据从存储引擎拉到服务器层，再根据查询条件过滤。在本书写作之际，MySQL5.6版本（未正式发布）包含了在存储引擎API上所做的一个重要的改进，其被称为**索引条件推送**（index condition pushdown）。这个特性将大大改善现在的查询执行方式，如此一来上面介绍的很多技巧也就不再需要了。

#### 使用索引扫描做排序

MySQL有两种方式可以生成有序的结果：通过**排序**操作；或者按**索引顺序扫描**；如果**EXPLAIN**出来的type列的值为**index**，则说明MySQL使用了索引扫描来做排序（不要和Extra列的Using index搞混淆了）。

扫描索引本身是很快的，因为只需要从一条索引记录移动到紧接着的下一条记录。但如果索引不能覆盖查询所需的全部列，那就不得不每扫描一条索引记录就都回表查询一次对应的行。这基本上都是随机I/O，因此按索引顺序读取数据的速度通常要比顺序地全表扫描慢，尤其是在I/O密集型的工作负载时。

MySQL可以使用同一个索引既满足排序，又用于查找行。因此，如果可能，设计索引时应该尽可能地同时满足这两种任务，这样是最好的。

只有当索引的列顺序和ORDER BY子句的顺序完全一致，并且所有列的排序方向（倒序或正序）都一样时，MySQL才能够使用索引来对结果做排序。如果查询需要关联多张表，则只有当ORDER BY子句引用的字段全部为第一个表时，才能使用索引做排序。ORDER BY子句和查找型查询的限制是一样的：需要满足索引的最左前缀的要求；否则，MySQL都需要执行排序操作，而无法利用索引排序。

有一种情况下ORDER BY子句可以不满足索引的最左前缀的要求，就是**前导列为常量**的时候。如果WHERE子句或者JOIN子句中对这些列指定了常量，就可以“弥补”索引的不足。

创建一个索引：
```
CREATE TABLE rental ( 
    ... 
    PRIMARY KEY (rental_id), 
    UNIQUE KEY rental_date (rental_date, inventory_id, customer_id), 
    KEY idx_fk_inventory_id (inventory_id), 
    KEY idx_fk_customer_id (customer_id), 
    KEY idx_fk_staff_id (staff_id), 
    ... 
);
```

能使用索引排序的情况：

使用常量，就可以从第二列开始排序：
```
WHERE rental_date = '2005-05-25' ORDER BY inventory_id DESC;
```

不能使用的情况：

不同的排序方向：
```
WHERE rental_date = '2005-05-25' ORDER BY inventory_id DESC, customer_id ASC;
```
引用不在索引中的列staff_id：
```
... WHERE rental_date=' 2005-05-25' ORDER BY inventory_id， staff_id;
```
第一列是范围条件：
```sql
WHERE rental_date > '2005-05-25' ORDER BY inventory_id, customer_id;
```
多个等于条件，相当于范围：
```sql
WHERE rental_date='2005-05-25' AND inventory_id IN(1, 2) ORDER BY customer_id; 
```

如果需要按不同方向做排序，一个技巧是存储该列值的反转串或者相反数。

#### 冗余和重复索引

MySQL允许在相同列上创建多个索引，这叫做重复索引，通常没理由这么做，除非是创建不同类型的索引来满足查询需求。

冗余索引：如果创建了索引（A，B），再创建索引（A）就是冗余索引，因为这只是前一个索引的前缀索引。

大部分情况下不需要冗余索引，但某些覆盖索引包含很长的列时，需要冗余索引。

增加新索引会导致INSERT、UPDATE、DELETE操作变慢。

#### 索引和锁

索引可以让查询锁定更少的行。InnoDB只有在访问行时才会加锁，所以索引可以减少锁的数量。

### 实战

现在需要看看哪些列拥有很多不同的取值，哪些列在WHERE子句中出现得最频繁。在有更多不同值的列上创建索引的选择性会更好。一般来说这样做都是对的。但有些列虽然选择性很低(比如性别)，但考虑到使用频率，还是建议作为前缀。

诀窍在于如果某个查询限制性别，可以在查询条件中新增AND SEX IN('m','f')来使用索引。

接下来需要考虑常见的WHERE组合。

使用不频繁的列可以忽略。

经常进行范围查询的列尽量放在最后面(比如age)，以便优化器使用尽可能多的索引列。

大数据集排序尽量使用索引排序：
```
SELECT<cols> FROM profiles WHERE sex='M' ORDER BY rating LIMIT 10;
```
可以建立(sex,rating)的索引。

另一个优化方式是使用延迟关联，先使用覆盖索引查询需要返回的主键，再根据主键返回需要的行：
```
SELECT<cols>
FROM profiles INNER JOIN(
    SELECT <primary key cols> 
    FROM profiles
    WHEREx.sex='M'
    ORDER BY rating LIMIT100000,10
    ) AS x USING(<primary key cols>);
```

### 更新索引统计信息

查看索引：
```
SHOW INDEX FROM table_name;
```

### 减少索引和数据的碎片

有三种数据碎片：行碎片，行间碎片，剩余空间碎片

使用OPTIMIZE TABLE或者导出再导入的方式来重新整理数据。

### 总结

有三个原则：

- 单行访问很慢。所以读取的块中能包含尽可能多所需要的行。
- 按顺序访问范围数据很快
- 覆盖索引查询很快，避免了大量的单行访问


 