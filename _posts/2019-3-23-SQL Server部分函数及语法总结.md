将最近使用的一些函数及语法，做个总结：

##### 字符串函数
- `LEN`(字符串) ：获取字符串长度
栗子：SELECT LEN('12345')   ====> 5
- `LEFT`(字符串,截取的长度)：截取左边字符串
- `RIGHT`(字符串,截取的长度)：截取右边字符串
栗子：SELECT RIGHT('12345',2)   ====> 45
- `LTRIM`(字符串)：清空字符串左边空格
- `RTRIM`(字符串)：清空字符串右边空格
栗子：SELECT LTRIM('    12 345')   ====> 12 345
- `STUFF`(字符串,需替换下标,需长度,替换为的字符串)：替换指定范围的字符串
栗子：SELECT STUFF('12345',2,1,'8')   ====> 18345
- `REPLACE`(字符串,要替换的字符串,替换为的字符串)：替换字符串
栗子：SELECT REPLACE(' 12345','1','8')   ====> 82345
- `UPPER`(字符串)，LOWER(字符串)：大小写转换
- `SUBSTRING`(字符串,下标,长度)：截取字符串
栗子：SELECT SUBSTRING('12345',2,2)   ====> 23
- `REPLICATE`(字符串,重复次数)：以指定次数重复字符串
栗子：SELECT REPLICATE('12345',2)   ====> 1234512345
- `REVERSE`(字符串)：反转字符串
栗子：REVERSE LTRIM('12345')   ====> 54321

##### 类型转换函数
- `CAST`(expression AS DateType)
栗子：CAST(11.11 AS INT)   ====> 11
            CAST(666 AS VARCHAR)   ====> '666'
- `CONVERT`(DateType, expression) 和CAST类似
栗子：CONVERT(INT, 11.11)====> 11
            CONVERT(VARCHAR, 666)====> '666'
- `CONVERT`(DateType, expression, style) 
style：日期格式样式，借以将 DATETIME 或 SMALLDATETIME 数据转换为字符数据；或者字符串格式样式，借以将 FLOAT、REAL、MONEY 或 SMALLMONEY 数据转换为字符数据。
栗子：CONVERT(NVARCHAR(32),GETDATE(),120)====> '2018-09-12 15:08:07'
CONVERT(NVARCHAR(32),1234.56,1)====> '1,234.56'

#### CASE语法
1.  CASE WHEN `exp1` THEN `result1` WHEN `exp2` THEN `result2` ELSE `result3` END 
2. CASE `exp` WHEN `value1` THEN `result1` WHEN `value2` THEN `result2` ELSE `result3` END 

#####操作符IN EXISES
- 如果子查询得出的结果集记录较少，主查询中的表较大且又有索引时应该用IN
 - 如果外层的主查询记录较少，子查询中的表大，又有索引时使用EXISTS
- 如果查询语句使用了NOT IN，那么对内外表都进行全表扫描，没有用到索引；而NOT EXISTS的子查询依然能用到表上的索引。所以无论哪个表大，用NOT EXISTS都比NOT IN要快。

##### 分页语法
- `TOP`
SELECT TOP(66) * FROM Table
- `ROW_NUMBER()`
SELECT * FROM (SELECT ROW_NUMBER OVER (ORDER BY xxx) AS ROW, * FROM Table) AS T WHERE T.ROW BETWEEN 1 AND 66
- `OFFSET` xxx `ROWS FETCH NEXT` yyy `ROWS ONLY`
SELECT * FROM Table OFFSET 1 ROWS FETCH NEXT 66 ROWS ONLY
- `LIMIT`
SELECT * FROM Table LIMIT 1,66

##### 复制表数据到另一个表
1. `INSERT INTO SELECT`语句：将表的数据插入到目标表，要求目标表是存在的。
在举栗子前，先创建测试数据：
```
DECLARE @TableA TABLE(
            A VARCHAR(10),
            B VARCHAR(10),
            C VARCHAR(10)
            )
DECLARE @TableB TABLE(
            B VARCHAR(10),
            C VARCHAR(10),
            D VARCHAR(10)
            )
INSERT INTO @TableA VALUES('a1','b1','c1')
INSERT INTO @TableA VALUES('a2','b2','c2')
INSERT INTO @TableA VALUES('a3','b3','c3')
```
栗子：
- 将@TableA表中的个别字段插入到目标表TableB中：
```
INSERT INTO @TableB(B,C) SELECT TA.A,TA.B FROM @TableA TA
SELECT * FROM @TableB
```
结果：
![bc](https://upload-images.jianshu.io/upload_images/2427856-3a9e4e1fa8877482.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 将@TableA表中的个别字段及常量变量插入到目标表TableB中：
```
INSERT INTO @TableB SELECT TA.A,TA.B,'d1' FROM @TableA TA
```
- 将@TableA表中的所有字段插入到目标表TableB中：
```
INSERT INTO @TableB SELECT * FROM @TableA
```

2. `SELECT INTO FROM`语句：和INSERT INTO SELECT功能一样但比INSERT INTO SELECT语句性能高，要求目标表是不存在的，执行该语句会创建目标表。批量转移数据至一个新表时，可以先用SELECT INTO，然后创建相关的索引，键，约束等。
栗子：
- 将@TableA表中的个别字段插入到目标表TableC
```
SELECT A,B INTO TableC FROM @TableA
```
- 将@TableA表中的所有字段插入到目标表TableC
```
SELECT * INTO TableC FROM @TableA
```
- 根据条件将@TableA表中的数据插入到目标表TableC
```
SELECT * INTO TABLEC FROM @TableA A WHERE A.A='a1'
```
3. `UPDATE SET FROM`语句：批量更新目标表。
测试数据：
```
DECLARE @TableD TABLE(
            D VARCHAR(10),
            E VARCHAR(10),
            F VARCHAR(10)
            )
INSERT INTO @TableD VALUES('a1','e1','f1')
INSERT INTO @TableD VALUES('d2','e2','f2')
INSERT INTO @TableD VALUES('d3','e3','f3')
```
栗子：
```
UPDATE @TableD SET D=TA.A,E=TA.B,F=TA.C FROM @TableA TA,@TableD TD WHERE TD.D=TA.A
```
结果：
![update](https://upload-images.jianshu.io/upload_images/2427856-01f25052b602e668.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
