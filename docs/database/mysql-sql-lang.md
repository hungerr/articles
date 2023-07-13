## MySQL简介及语法

### 基本概念

- **数据库**(database)： 保存有组织的数据的容器(通常是一个文件或一组文件)
- **表**(table)：某种特定类型数据的结构化清单，可以想象成一个网格
- **模式**(schema)：关于数据库和表的布局及特性
- **行**(row)：表中的一条记录
- **列**(column)：每一列存储着一条特定的信息
- **主键**(primary key)：每一行中唯一标识自己的一列(或者一组列)，任意两行都不能有相同的主键值，不能为空
- **外键**(foreignkey)：外键为某个表中的一列，它包含另一个表的主键，定义了两个表之间的关系
- **SQL**：结构化查询语言

### MySQL

- 连接：`mysql -u username -pPassword -P 3306 -h host`
- 使用数据库：`USE database;`
- 查看数据库：`SHOW DATABASES;`
- 查看表：`SHOW TABLES;`
- 查看列：`SHOW COLUMNS FROM tablename;`或者`DESCRIBE tablename;`
- 查看服务器状态：`SHOW STATUS;`
- 查看创建数据库或者表语句：`SHOW CREATE DATABASE/TABLE;`
- 查看用户权限：`SHOW GRANTS;`

### 检索数据SELECT

- 检索单行：`SELECT prod_name FROM products;`
- 检索多行：`SELECT prod_name,title FROM products;`
- 去重：`SELECT DISTINCT prod_name FROM products;`
- 所有行：`SELECT * FROM products;`
- 限制行数：`SELECT prod_name FROM products LIMIT 5;`,`LIMIT 5,10`指从第5行开始的10行，行从0开始

### 排序 ORDER BY

- 多行排序：`SELECT prod_name FROM products ORDER BY title,name;`
- 降序： `SELECT prod_name FROM products ORDER BY title DESC,name;`，默认升序，DESC跟随列名，多列需要多个DESC

### 过滤数据WHERE

- 基本语法：`SELECT prod_name FROM products WHERE name = 'name';`
- 子句：`=,<>,!=,<,<=,>,>=,BETWEEN`，默认不区分大小写，单引号限定字符串
- 范围检查：`BETWEEN 5 AND 10`
- 空值：`WHERE name IS NULL`
- 组合：`AND OR`，`WHERE id = 10 OR WHERE id = 20`，用圆括号明确分组
- **IN**：`WHERE id IN (2001,2002)`，类似于OR，但更清楚直观，更快，还可以包含其它SELECT语句。
- **NOT**：否定后面的条件，`WHERE id NOT IN (2001,2002)`
- 可以对列名，表达式或者聚合数据进行筛选

### 通配符LIKE

- **%**：表示任意字符出现任意次数(0或更多)，`WHERE name LIKE 'jet%';`，不匹配NULL
- `_`：只匹配单个字符，`WHERE name LIKE '_ jet'`
- 技巧：不要过度使用通配符，能不用则不用，尽量不要把通配符放在搜索模式的开始处

### 正则表达式

- **REGEXP**：`WHERE name REGEXP 'jet';`，**LIKE**匹配整个列，而**REGEXP**在**列内值匹配**
- OR匹配：`WHERE name REGEXP '1000|2000';`
- 匹配几个之一： `[]`，`WHERE name REGEXP '[jet]';`匹配jet之一，排除，`[^123]`匹配非123之一，`[0-9]`，`[a-z]`
- 特殊字符：`\\.[]-`

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/database/images/regexp-char.png)

`^`有双重意义，开始或者非。`$`代表结尾。

### 计算字段

- 拼接字段Concat：`SELECT Concat( vend_ name, ' (', vend_ country, ')') FROM vendors ORDER BY vend_name;`
- **AS**：将计算字段重命名
- 算数计算：`+-*/`

### 函数

**文本处理函数**

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/database/images/text-func.png)

**日期和时间处理函数**

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/database/images/date-func.png)

日期格式必须为`yyyy-mm-dd`，`WHERE order_date = '2005-09-01';`

比如，存储的order_date值为`2005-09-0111:30:05`，则`WHERE order_date = '2005-09-01'`失败。 即使给出具有该日期的一行，也不会把它 检索出来， 因为 WHERE 匹配失败。需要使用日期函数，`WHERE Date(order_date) = '2005-09-01';`

查询9月份的方法：`WHERE Date(order_date) BETWEEN '2005-09-01' AND '2005-09-30';`或者`WHERE Year(order_date) = 2005 AND Month(order_date) = 9;`

**数值处理函数**

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/database/images/num-func.png)

### 汇总聚合函数

- **AVG**：对表中行数计数求和计算平均值，忽略NULL行
- **COUNT**：使用COUNT(*)对所有行计数，包括NULL行，使用COUNT(column)对特定行计数，不包括NULL行
- **MAX**：要求指定列名，返回指定列的最大值。忽略NULL行。用于文本数据时，如果数据按照相应的列排序，则返回最后一行
- **MIN**：要求指定列名，返回指定列的最小值。忽略NULL行。用于文本数据时，如果数据按照相应的列排序，则返回最前面的一行
- **SUM**：求指定列的和。也可以用来合计计算值。

聚集函数聚集不同值得时候，使用**DISTINCT**，`SELECT AVG(DISTINCT PRICE)`。DISTINCT只能用于列名，不能用于计算或者表达式。用于MAX或者MIN无价值。

### 分组数据GROUP

`SELECT vend_id, COUNT(*) AS num_prods FROM products GROUP BY vend_id;`

GROUP BY子句指示MySQL分组数据，然后对每个组而不是整个结果集进行聚集。

规定：

- GROUP BY子句可以包含任意数量的列，这使得能对分组进行嵌套。
- 如果在GROUP BY子句中嵌套了分组，数据将在最后规定的分组上进行汇总。换句话说，在建立分组时，指定的所有列都一起计算（ 所以不能 从个别的列取回数据）。 
- GROUP BY子句中的每个列都必须是检索列或有效的表达式(但不能是聚集函数)。如果要在SELECT中使用表达式，必须是在GROUP BY中指定的相同的表达式，不能是别名
- 除聚集语句外，SELECT语句中的每个列都必须在GROUP BY子句中给出
- 如果分组列中有NULL，NULL将作为一个分组返回。
- GROUP BY语句必须出现在WHERE语句之后，ORDER BY语句之前
- 过滤分组使用**HAVING**（支持WHERE的所有操作），WHERE在数据分组前进行过滤，HAVING在数据分组后进行过滤

### 子查询

子查询即嵌套在其它查询中的查询

1.可以把一条SELECT语句返回的结果用于另一条SELECT语句的WHERE子句:
```
SELECT cust_id 
FROM orders 
WHERE order_num IN (SELECT order_num 
                    FROM orderitems 
                    WHERE prod_id = 'TNT2'); 
```
子查询总是从内到外查询。

在WHERE子句中使用子查询，应该保证SELECT语句具有与WHERE子句中相同数目的列。

子查询一般与IN操作符结合使用，但也可以用于测试等于(=)、不等于(<)等。

2.计算字段也可以用做子查询 
```
SELECT cust_name, 
       cust_state, 
       (SELECT COUNT(*) 
        FROM orders 
        WHERE orders.cust_id = customers.cust_id) AS orders 
FROM customers 
ORDER BY cust_name;
```
涉及到外部查询的子查询称为相关子查询。使用了完全限定的列名。

### 联结表join

联结是一种机制，用来在一条SELECT语句中关联表。

使用WHERE子句联结表：
```
SELECT vend_name, prod_name, prod_price 
FROM vendors, products 
WHERE vendors. vend_id = products.vend_id 
ORDER BY vend_name, prod_name;
```
由没有联结条件的表返回的结果为笛卡尔积。数目为第一个表中的行数乘以第二个表中的行数。

1.**内部联结**

上述称之为等值联结，它基于两个表之间的相等测试。也称之为内部联结。可以使用新的语法，使用ON代替WHERE：
```
SELECT vend_name, prod_name, prod_price 
FROM vendors INNER JOIN products 
ON vendors.vend_id = products.vend_id;
```
2.**外部联结**

外部联结包含了那些在相关表中没有关联行的行。
```
SELECT customers.cust_id, orders.order_num 
FROM customers LEFT OUTER JOIN orders 
ON customers.cust_id = orders.cust_id;
```
LEFT指OUTER JOIN左边的表，RIGHT指OUTER JOIN右边的表。

3.**自联结**

自联结通常用作外部语句用来代替从相同表中检索数据时使用的子查询语句。

子查询：
```
SELECT prod_id, prod_name 
FROM products 
WHERE vend_id = (SELECT vend_id FROM products WHERE prod_id = 'DTNTR');
```
自联结：
```
SELECT p1. prod_id, p1.prod_name 
FROM products AS p1, products AS p2 
WHERE p1.vend_id = p2.vend_id AND p2.prod_id = 'DTNTR';
```

4.**自然联结**

自然联结排除多次出现，使每个列只返回一次。一般是通过对表使用通配符(SELECT *)，对所有其他表的列使用明确的子集来完成的。

可以在联结中使用聚合函数：
```
SELECT customers.cust_name, 
       customers.cust_id,  
       COUNT(orders.order_num) AS num_ord 
FROM customers INNER JOIN orders 
ON customers.cust_id = orders.cust_id 
GROUP BY customers.cust_id;
```

### 组合查询UNION

MySQL允许执行多个查询，并将结果作为单个查询结果集返回。

UNION必须由两条或以上的SELECT语句组成，语句之间用UNION分隔。

UNION中的每个查询必须包含相同的列，表达式或聚集函数(顺序可以不同)

列数据必须兼容，类型不必完全相同，但必须包含DBMS可以隐式转换的类型。

UNION类似于WHERE OR语句，UNION不包含重复的行，UNION ALL 包括重复的行

只能包含一条ORDER BY语句，出现在最后

### 全文搜索

MyISAM支持全文搜索，InnoDB不支持

使用全文搜索时，MySQL不需要分别查看每个行，不需要分别分析和处理每个词。MySQL创建指定列中各词的一个索引，搜索可以针对这些词进行。

创建全文搜索：
```
CREATE TABLE productnotes 
( 
  note_id int NOT NULL AUTO_INCREMENT, 
  prod_id char(10) NOT NULL, 
  note_date datetime NOT NULL, 
  note_text text NULL , 
  PRIMARY KEY(note_id), 
  FULLTEXT(note_text) 
) 
ENGINE= MyISAM;
```

进行全文搜索：
```
SELECT note_text FROM productnotes WHERE Match(note_text) Against('rabbit');
```

3个或3个以下字符的短词被忽略且从索引中排除。

内建的非用词stopword列表会被忽略。

一个词出现在50%以上的行中，则将它作为一个非用词忽略。

忽略词中的单引号。

### 插入数据INSERT

插入完整的行：
```
INSERT INTO Customers 
VALUES( NULL, 'Pep E. LaPew', '100 Main Street', 'Los Angeles', 'CA', '90046', 'USA', NULL, NULL);
```

必须按照次序，NULL被MySQL忽略。

可以指定列名：
```
INSERT INTO 
customers( cust_name, cust_address, cust_city, cust_state, cust_zip, cust_country, cust_contact, cust_email) 
VALUES('Pep E.LaPew', '100 Main Street','Los Angeles', 'CA', '90046', 'USA', NULL, NULL);
```

省略的行必须满足 允许NULL值或者给出默认值。

插入多个行用逗号隔开：
```
INSERT INTO
VALUES(),
();
```

插入检索出的数据：
```
INSERT INTO 
customers(cust_id, cust_contact, cust_email, cust_name, cust_address, cust_city, cust_state, cust_zip, cust_country) 
SELECT cust_id, cust_contact, cust_email, cust_name, cust_address, cust_city, cust_state, cust_zip, cust_country 
FROM custnew;
```
不要求列名匹配，使用的是位置。

### 更新数据UPDATE

```
UPDATE customers 
SET cust_email = 'elmer@ fudd.com' 
WHERE cust_id = 10005;
```
可以设置为NULL值，以达到删除目的。

使用WHERE避免全部更新

### 删除数据DELETE
```
DELETE FROM customers 
WHERE cust_id = 10006;
```

### 创建表CREATE TABLE
```
CREATE TABLE customers 
( 
  cust_id int NOT NULL AUTO_INCREMENT, 
  cust_name char(50) NOT NULL , 
  cust_address char(50) NULL , 
  PRIMARY KEY (cust_id) 
) ENGINE= InnoDB;
```
创建表时，表名必须不存在。如果仅想在一个表不存在时创建它，应该在表名后给出IF NOT EXISTS。

列默认是NULL

使用AUTO_INCREMENT自增长，每个表只允许一个，必须被索引。获取这个值：
```
SELECT last_insert_id();
```
这个值也可以设定，后续的增量从新设值开始。

指定默认值：DEFAULT

### 创建索引：
```
CREATE INDEX indexname 
ON tablename (column [ASC| DESC], ...);
```

### 更新表ALTER TABLE

语法：
```
ALTER TABLE tablename 
( 
  ADD column datatype [NULL| NOT NULL] [CONSTRAINTS], 
  CHANGE column columns datatype [NULL| NOT NULL] [CONSTRAINTS], 
  DROP column, 
  ... 
);

```
添加列：
```
ALTER TABLE vendors 
ADD vend_phone CHAR(20);
```

删除列：
```
ALTER TABLE Vendors 
DROP COLUMN vend_phone;
```

添加外键：
```
ALTER TABLE orderitems 
ADD CONSTRAINT fk_orderitems_orders 
FOREIGN KEY (order_num) 
REFERENCES orders (order_num); 
```

### 删除表DROP

`DROP TABLE custom;`

### 重命名

`RENAME TABLE customers2 TO customers;`

### 视图

视图是虚拟的表，视图只包含动态检索数据的查询

创建视图，视图必须唯一命名：
```
CREATE VIEW productcustomers AS 
SELECT cust_name, cust_contact, prod_id 
FROM customers, orders, orderitems 
WHERE customers.cust_id = orders.cust_id 
AND orderitems.order_num = orders.order_num;
```
查看创建视图的语句：
```
SHOW CREATE VIEW viewname;
```
删除视图：
```
DROP VIEW viewname;
```
先DROP再用CREATE：
```
CREATE OR REPLACE VIEW;
```
重新格式化查询数据：
```
CREATE VIEW vendorlocations AS 
SELECT Concat( RTrim( vend_name), ' (', RTrim(vend_country), ')') AS vend_title 
FROM vendors 
ORDER BY vend_name;
```
过滤数据：
```
CREATE VIEW customeremaillist AS 
SELECT cust_id, cust_name, cust_email 
FROM customers 
WHERE cust_email IS NOT NULL;
```
也可以计算字段。

通常，视图是可以更新的，更新一个视图将更新基表。视图主要用于数据检索，更新很少。使用分组、联结、子查询、并、聚集函数、DISTINCT
、导出计算列时不能更新。

### 存储过程

存储过程就是为以后的使用而保存的一条或多条MySQL语句的集合。可视为批文件。保证了数据的完整性。

执行存储过程：
```
CALL productpricing(@ pricelow, 
                    @pricehigh, 
                    @priceaverage);
```

创建存储过程，参数放在()中，可以为空：
```
CREATE PROCEDURE productpricing() 
BEGIN 
    SELECT Avg( prod_ price) AS priceaverage 
    FROM products; 
END;
```
语句内使用了分隔符`;`，可以替代；
```
DELIMITER // 

CREATE PROCEDURE productpricing() 
BEGIN 
    SELECT Avg( prod_ price) AS priceaverage 
    FROM products; 
END // 

DELIMITER ;
```
删除存储过程：
```
DROP PROCEDURE productpricing;
```
使用变量：
```
CREATE PROCEDURE productpricing( 
    OUT pl DECIMAL(8, 2), 
    OUT ph DECIMAL(8, 2), 
    OUT pa DECIMAL(8, 2) 
) 
BEGIN 
    SELECT Min(prod_price) 
    INTO pl 
    FROM products; 
    SELECT Max(prod_price) 
    INTO ph 
    FROM products; 
    SELECT Avg(prod_price) 
    INTO pa 
    FROM products; 
END;
```
此存储过程接收三个参数，MySQL支持IN(传递给存储过程)、OUT(从存储过程传出)、INOUT(对存储过程传入和传出)。SELECT用来检索值，然后保存到相应的变量(INTO指定)

调用存储过程，存储：
```
CALL productpricing(@pricelow, 
                    @pricehigh, 
                    @priceaverage);
```
MySQL变量都要以@开头。

显示：
```
SELECT @priceaverage;
SELECT @pricehigh, @pricelow, @priceaverage;
```
使用IN传递进参数：
```
CREATE PROCEDURE ordertotal( 
    IN onumber INT, 
    OUT ototal DECIMAL(8,2) 
) 
BEGIN 
    SELECT Sum(item_price*quantity) 
    FROM orderitems 
    WHERE order_num = onumber 
    INTO ototal; 
END;
```
调用：
```
CALL ordertotal(20005, @total);
SELECT @total;
```
智能存储：
```
-- Name: ordertotal 
-- Parameters: onumber = order number 
--             taxable = 0 if not taxable, 1 if taxable 
--             ototal = order total variable 
CREATE PROCEDURE ordertotal(
    IN onumber INT, 
    IN taxable BOOLEAN,
    OUT ototal DECIMAL(8, 2) 
) COMMENT 'Obtain order total, optionally adding tax' BEGIN 

    -- Declare variable for total 
    DECLARE total DECIMAL(8, 2); 
    -- Declare tax percentage 
    DECLARE taxrate INT DEFAULT 6; 

    -- Get the order total 
    SELECT Sum(item_price*quantity) 
    FROM orderitems 
    WHERE order_num = onumber 
    INTO total; 

    -- Is this taxable? 
    IF taxable THEN 
        -- Yes, so add taxrate to the total 
        SELECT total+( total/ 100* taxrate) INTO total; 
    END IF; 

    -- And finally, save to out variable 
    SELECT total INTO ototal; 

END;
```
增加了注释(--)。DECLARE指定变量名和数据类型，定义了两个局部变量。IF检测条件。

使用：
```
CALL ordertotal(20005, 0, @total); 
SELECT @total;
```
检查存储过程：
```
SHOW CREATE PROCEDURE ordertotal;
SHOW PROCEDURE STATUS LIKE 'ordertotal';
```

### 游标cursor

MySQL检索操作返回一组称为结果集的行。有时候需要在结果集中前进或者后退一行，这是使用游标的原因。MySQL游标只能用于存储过程和函数。

首先声明游标，只是定义要使用的SELECT语句，打开游标使用(执行SELECT语句)，取出检索各行，关闭游标。

创建游标：
```
CREATE PROCEDURE processorders() 
BEGIN 
    DECLARE ordernumbers CURSOR 
    FOR 
    SELECT ordernum FROM orders; 
END;
```
使用游标：
```
OPEN ordernumbers;
```
关闭游标：
```
CLOSE ordernumbers;
```
声明过的游标不需要再声明，关闭后重新打开即可。

取出数据做处理：
```
CREATE PROCEDURE processorders() 
BEGIN 

    -- Declare local variables 
    DECLARE done BOOLEAN DEFAULT 0; 
    DECLARE o INT; 
    DECLARE t DECIMAL(8, 2); 

    -- Declare the cursor 
    DECLARE ordernumbers CURSOR 
    FOR 
    SELECT order_num FROM orders; 

    -- Declare continue handler 
    DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET done= 1; 

    -- Create a table to store the results 
    CREATE TABLE IF NOT EXISTS ordertotals (order_num INT, total DECIMAL(8, 2)); 

    -- Open the cursor 
    OPEN ordernumbers;
 
    -- Loop through all rows
    REPEAT 

        -- Get order number 
        FETCH ordernumbers INTO o; 

        -- Get the total for this order 
        CALL ordertotal(o, 1, t); 

        -- Insert order and total into ordertotals 
        INSERT INTO ordertotals(order_num, total) VALUES(o, t); 

    -- End of loop 
    UNTIL done END REPEAT; 

    -- Close the cursor 
    CLOSE ordernumbers; 

END;
```

### 触发器

可以使用触发器使某条语句在事件发生时自动执行。

触发器可以响应语句：`DELETE`、`INSERT`、`UPDATE`

创建触发器需要给出信息：唯一名称、表名、响应的活动，何时执行(之前或之后)：
```
CREATE TRIGGER newproduct AFTER INSERT ON products 
FOR EACH ROW SELECT 'Product added';
```
触发器按每个表每个事件每次地定义，每个表每个事件每次只允许一次触发器。每个表最多支持6个触发器(三个响应事件之前之后)

删除触发器：
```
DROP TRIGGER newproduct; 
```
INSERT触发器，可以引用一个NEW的虚拟表，访问被插入的行，BEFORE操作中，可以更新NEW中的值。对于AUTO_INCREMENT，NEW在INSERT执行之前包含0，之后包含新的自动生成值。
```
CREATE TRIGGER neworder AFTER INSERT ON orders FOR EACH ROW SELECT NEW.order_num;
```
BEFORE用于数据验证和净化

DELETE触发器，可以引用OLD表，访问被删除的行。将被删除的订单存档：
```
CREATE TRIGGER deleteorder BEFORE DELETE ON orders 
FOR EACH ROW 
BEGIN 
    INSERT INTO archive_orders(order_num, order_date, cust_id) VALUES(OLD.order_num, OLD.order_date, OLD.cust_id); 
END;
```
使用BEFORE，若果不能存档，则DELETE操作将被放弃

UPDATE触发器可以引用OLD和NEW，BEFORE中NEW可以被更新，OLD只读
```
CREATE TRIGGER updatevendor BEFORE UPDATE ON vendors 
FOR EACH ROW SET NEW.vend_state = Upper(NEW.vend_state);
```

### 事务

开始事务：
```
START TRANSACTION
```
回退：
```
ROLLBACK;
```
只能回退INSERT UPDATE DELETE语句，不能回退CREATE DROP

提交：
```
COMMIT;
```
使用保留点：
```
SAVEPOINT delete1;
ROLLBACK TO delete1;
```
默认MySQL行为自动提交所有的更改。改变：
```
SET autocommit= 0; 
```

### 字符集

查询：
```
SHOW CHARACTER SET;
SHOW COLLATION;
SHOW VARIABLES LIKE 'character%'; 
SHOW VARIABLES LIKE 'collation%';
```
创建，表：
```
CREATE TABLE mytable 
( 
    columnn1 INT, 
    columnn2 VARCHAR( 10) 
) DEFAULT CHARACTER SET hebrew COLLATE hebrew_general_ci;
```
只指定character时，使用字符集及其默认的校对

单独设定列：
```
CREATE TABLE mytable 
( 
    columnn1 INT, 
    columnn2 VARCHAR( 10), 
    column3 VARCHAR( 10) CHARACTER SET latin1 COLLATE latin1_ general_ ci 
)
```
排序：
```
SELECT * FROM customers 
ORDER BY lastname, firstname COLLATE latin1_general_cs;
```

### 用户权限管理

查看用户：
```
USE mysql; 
SELECT user FROM user;
```
创建用户：
```
CREATE USER ben IDENTIFIED BY 'p@$$w0rd';
```
重命名用户：
```
RENAME USER ben TO bforta;
```
删除用户：
```
DROP USER bforta;
```
查看权限：
```
SHOW GRANTS FOR bforta;
```
设置权限，要给出权限、表或数据库、用户名：
```
GRANT SELECT,INSERT ON crashcourse.* TO beforta; 
```
撤销权限：
```
REVOKE SELECT ON crashcourse.* FROM beforta;. 
```
层次：
 
- 整个服务器：GRANT ALL
- 整个数据库： ON database.*
- 特定表： ON database.table

更改口令：
```
SET PASSWORD FOR bforta = Password('n3w p@$$w0rd'); 
```
设置自己的口令：
```
SET PASSWORD = Password(' n3w p@$$ w0rd');
```