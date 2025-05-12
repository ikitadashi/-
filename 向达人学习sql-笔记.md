“导入” CASE命令的构造
CASE分为单纯CASE（simple case）和检索CASE（searched case）
书写方法
```sql
-- 単純CASE式
CASE sex
  WHEN '1' THEN '男'
  WHEN '2' THEN '女'
 ELSE 'その他' END
```
```sql
-- 検索CASE式
CASE WHEN sex = '1' THEN '男'
     WHEN sex = '2' THEN '女'
 ELSE 'その他' END
```
在sex这一列中，如果1就替换为男，2替换为女。
注意：当满足一个when就进行替换，会无视下面的when。具有排他性
最后要写else .... end;
虽然else不写也没关系，但是以防万一，防止出现空值，最好写一下
通常，CASE和其他命令一起使用

通过code1_1创建表格PopTbl，然后通过CASE选择pref_name这一列，对里面进行分类。然后先进行求和，再通过分类输出表格。
```sql
-- 県名を地方名に再分類する
SELECT CASE pref_name
              WHEN '徳島' THEN '四国'
              WHEN '香川' THEN '四国'
              WHEN '愛媛' THEN '四国'
              WHEN '高知' THEN '四国'
              WHEN '福岡' THEN '九州'
              WHEN '佐賀' THEN '九州'
              WHEN '長崎' THEN '九州'
       ELSE 'その他' END AS district,
       SUM(population)
  FROM PopTbl
 GROUP BY CASE pref_name
              WHEN '徳島' THEN '四国'
              WHEN '香川' THEN '四国'
              WHEN '愛媛' THEN '四国'
              WHEN '高知' THEN '四国'
              WHEN '福岡' THEN '九州'
              WHEN '佐賀' THEN '九州'
              WHEN '長崎' THEN '九州'
          ELSE 'その他' END;
```
输出结果
```
district     SUM(population)
その他       450
九州         600
四国         650
```
第二种写法
```sql
SELECT CASE pref_name
              WHEN '徳島' THEN '四国'
              WHEN '香川' THEN '四国'
              WHEN '愛媛' THEN '四国'
              WHEN '高知' THEN '四国'
              WHEN '福岡' THEN '九州'
              WHEN '佐賀' THEN '九州'
              WHEN '長崎' THEN '九州'
       ELSE 'その他' END AS district,
       SUM(population)
  FROM PopTbl
 GROUP BY district;
```
直接通过district这一列来进行group by，一样的输出结果。（注意，oracle和Db2 sqlsever这么写会报错）


PopTbl2进行各地性别人口分类
使用case命令简化
```sql
SELECT pref_name,
       -- 男性の人口
       SUM( CASE WHEN sex = '1' THEN population ELSE 0 END)   --sum求和，当sex这一列为1的进行求和，population进行求和
AS 男性,
       -- 女性の人口
       SUM( CASE WHEN sex = '2' THEN population ELSE 0 END) 
AS 女性
  FROM PopTbl2
 GROUP BY pref_name;
```
注意，如果不进行group by的话就会分开输出，男的一行，女的一行。使用group by合并在一行输出。


不同条件下进行update
先创建名为Salaries的表
```sql
CREATE TABLE Salaries
(name varchar(10) PRIMARY KEY,
salary int
); 

INSERT INTO Salaries VALUES ('相田', 300000);
INSERT INTO Salaries VALUES ('神崎', 270000);
INSERT INTO Salaries VALUES ('木村', 220000);
INSERT INTO Salaries VALUES ('斉藤', 290000);
```
表格如下

|name |    salary  |
| --- | --- |
|相田  |   300000  |
|神崎  |  270000   |
|木村  |   220000  |
|斉藤  |  290000   |

要求：
工资高于30万的，减少30%
工资大于等于25万小于等于28万的，增加20%

如果如下写的话
```sql
-- 条件1
 UPDATE Personnel
 SET salary = salary * 0.9
 WHERE salary >= 300000;
-- 条件2
 UPDATE Personnel
 SET salary = salary * 1.2
 WHERE salary >= 250000 AND salary < 280000;
```
那么，输出结果就会是
|name  |   salary  |
| --- | --- |
|相田  |   324000 |  
|神崎  |   270000 |
|木村  |   220000 |
|斉藤   |  290000  |

相田那行计算错误，因为先执行了300000x0.9，然后又执行了x1.2

正确执行如下
```sql
UPDATE Salaries
 SET salary = CASE WHEN salary >= 300000
 THEN salary * 0.9
 WHEN salary >= 250000 AND salary < 280000
 THEN salary * 1.2
 ELSE salary END;
```
注意：最后的 ELSE salary非常重要。默认ELSE NULL。如果不写的话，那会不满足条件1或2的人直接输出NULL 空值。
可以通过这个方法通过一次UPDATE多个主key，避免多次写入。
```sql
UPDATE SomeTable
   SET p_key = CASE WHEN p_key = 'a'
                    THEN 'b'
                    WHEN p_key = 'b'
                    THEN 'a'
               ELSE p_key END
 WHERE p_key IN ('a', 'b');
```
这个方法在PostgreSQL和MySQL中不能使用，会报错。


表格之间对比
创建CourseMaster和OpenCourses表，整理每个月需要开设的讲座
```sql
-- テーブルのマッチング：EXISTS述語の利用
SELECT CM.course_name,
       CASE WHEN EXISTS   -- 当存在时
                  (SELECT course_id FROM OpenCourses OC     -- 把OpenCourses表简写成OC
                    WHERE month = 201806
                      AND OC.course_id = CM.course_id) THEN '○'    -- 当表CourseMaster中course_id和OpenCourses中相同时，输出⚪
            ELSE '×' END AS "6 月",  -- 不是的话输出× 列为6月
       CASE WHEN EXISTS 
                  (SELECT course_id FROM OpenCourses OC
                    WHERE month = 201807
                      AND OC.course_id = CM.course_id) THEN '○'
            ELSE '×' END AS "7 月",
       CASE WHEN EXISTS
                  (SELECT course_id FROM OpenCourses OC
                    WHERE month = 201808
                      AND OC.course_id = CM.course_id) THEN '○'
            ELSE '×' END AS "8 月"
  FROM CourseMaster CM;  -- 把CourseMaster简写成CM
```
使用EXISTS命令可以更有效率处理，当表格增加月数了也只需要修改SELECT即可。

在CASE中使用集约函数
要求
1，只属于1个俱乐部的学生，导出其所属的俱乐部ID
2，对于属于多个俱乐部的学生，导出其主要俱乐部ID
```sql
SELECT std_id,
       CASE WHEN COUNT(*) = 1  -- 1つのクラブに専念する学生の場合
            THEN MAX(club_id)
       ELSE MAX(CASE WHEN main_club_flg = 'Y'  --一个学生属于多个俱乐部的情况，查看main_club_flg 是否为Y，提取是Y的那一行的club_id
                     THEN club_id
                ELSE NULL END) END AS main_club  -- 都没有的就为空，然后把收集的club_id输出到main_club
  FROM StudentClub
 GROUP BY std_id; -- 由GROUP BY进行约束
```
通过CASE WHEN COUNT(*) = … ELSE… 来进行判断“属于一个俱乐部还是多个俱乐部”

练习题
1，提取出多个列中的最大值
```sql
-- xとyとzの最大値
SELECT key,
       CASE WHEN CASE WHEN x < y THEN y ELSE x END < z
            THEN z
            ELSE CASE WHEN x < y THEN y ELSE x END
        END AS greatest
  FROM Greatests;
```
还有一种写法
```sql
-- 提取每个 key 的最大值（无 GREATEST 函数）
SELECT 
    key,
    CASE 
        WHEN x >= y AND x >= z THEN x
        WHEN y >= x AND y >= z THEN y
        ELSE z
    END AS 最大值
FROM Greatests;

-- 如果有GREATEST函数的话
SELECT key, GREATEST(GREATEST(x,y), z) AS greatest
 FROM Greatests;
```
2.求和并修改行名输出
```sql
SELECT sex,
       SUM(population) AS total,
       SUM(CASE WHEN pref_name = '徳島' 
                THEN population ELSE 0 END) AS tokushima,
       SUM(CASE WHEN pref_name = '香川' 
                THEN population ELSE 0 END) AS kagawa,
       SUM(CASE WHEN pref_name = '愛媛' 
                THEN population ELSE 0 END) AS ehime,
       SUM(CASE WHEN pref_name = '高知' 
                THEN population ELSE 0 END) AS kouchi,
       SUM(CASE WHEN pref_name IN ('徳島', '香川', '愛媛', '高知')
                THEN population ELSE 0 END) AS saikei
  FROM PopTbl2
 GROUP BY sex;
-- ORDER BY sex;  -- 如果需要按照性别排序的话
```
代码功能

SUM(population)：
统计每个性别的总人口数。

SUM(CASE WHEN pref_name = '徳島' THEN population ELSE 0 END)：
统计徳島的性别人口数，类似的逻辑用于香川、愛媛、高知。

SUM(CASE WHEN pref_name IN ('徳島', '香川', '愛媛', '高知') THEN population ELSE 0 END)：
统计四国地区（徳島、香川、愛媛、高知）的性别人口总数。

GROUP BY sex：
按性别分组，分别统计男性和女性的数据。


窗口函数 window functions  
使用场景：  
排名问题：每个部门按业绩来排名  
topN问题：找出每个部门排名前N的员工进行奖励  
复购分析：App内要分析复购用户有多少  
累计问题：医院要经常统计累计患者数  

例子：求商品的移动平均 表为Shohin
```sql
SELECT shohin_id, shohin_mei, hanbai_tanka,
       AVG (hanbai_tanka) OVER (ORDER BY shohin_id                         
                                ROWS BETWEEN 2 PRECEDING                   
                                         AND CURRENT ROW) AS moving_avg    
  FROM Shohin;
```
功能说明  
AVG(hanbai_tanka) OVER (...):  
使用窗口函数计算 hanbai_tanka 的移动平均值。  
ORDER BY shohin_id:  
按 shohin_id 的顺序定义窗口的计算顺序。  
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW:  
定义窗口范围为当前行及其前两行。  
AS moving_avg:  
将计算结果命名为 moving_avg，表示移动平均值。  

解释：  
对于 shohin_id = 1，只有当前行可用，因此 moving_avg = hanbai_tanka_1。  
对于 shohin_id = 2，窗口范围是第 1 行和第 2 行，因此 moving_avg = (hanbai_tanka_1 + hanbai_tanka_2) / 2   
对于 shohin_id = 3，窗口范围是第 1 行到第 3 行，因此 moving_avg = (hanbai_tanka_1+hanbai_tanka_2+hanbai_tanka_3) / 3   
以此类推。
关于移动平均具体可以参考
https://zhuanlan.zhihu.com/p/151786842
https://www.srush.co.jp/blog/1014712986

也可以这么写，更明确的显示window定义
```sql
SELECT shohin_id, shohin_mei, hanbai_tanka,
       AVG(hanbai_tanka) OVER W AS moving_avg
  FROM Shohin
 WINDOW W AS (ORDER BY shohin_id
                 ROWS BETWEEN 2 PRECEDING
                          AND CURRENT ROW);
```
功能说明  
1。SELECT 子句：  
选择的列包括：  
shohin_id：商品的唯一标识符。  
shohin_mei：商品名称。  
hanbai_tanka：商品的销售单价。  
AVG(hanbai_tanka) OVER W：基于窗口 W 计算的移动平均值，命名为 moving_avg。  

2.WINDOW 子句：定义了一个窗口 W，用于窗口函数的计算。  
ORDER BY shohin_id：按 shohin_id 的顺序排列数据，窗口函数的计算将基于此顺序。  
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW：定义窗口范围为当前行及其前两行。  

3.AVG(hanbai_tanka) OVER W：使用窗口函数 AVG 计算窗口范围内 hanbai_tanka 的平均值。  

4.FROM Shohin：数据来源于 Shohin 表。  

执行逻辑  
按 shohin_id 排序：数据会按照 shohin_id 的顺序排列。  
定义窗口范围：对于每一行，窗口范围包括当前行及其前两行。  
计算移动平均值：在窗口范围内，计算 hanbai_tanka 的平均值。  

命名的window函数，可以重复使用window  
```sql
SELECT 
    shohin_id, 
    shohin_mei, 
    hanbai_tanka,
    AVG(hanbai_tanka)   OVER W AS moving_avg,  --计算窗口范围内的平均值，命名为 moving_avg。
    SUM(hanbai_tanka)   OVER W AS moving_sum,  --计算窗口范围内的总和，命名为 moving_sum。
    COUNT(hanbai_tanka) OVER W AS moving_count, --计算窗口范围内的记录数，命名为 moving_count。
    MAX(hanbai_tanka)   OVER W AS moving_max --计算窗口范围内的最大值，命名为 moving_max
FROM Shohin
WINDOW W AS (
    ORDER BY shohin_id  --按 shohin_id 的顺序排列数据，窗口函数的计算将基于此顺序。
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW -- 定义窗口范围为当前行及其前两行。
);
从第3行开始，moving_count为3，moving_avg=（当前商品价格+前2行商品价格）/3
```
1张图了解window函数的功能  
<img width="297" alt="image" src="https://github.com/user-attachments/assets/95d5f3f3-0a01-4981-ad7b-dd8d0bdd8d55" />

1.PARTITION BY语句来分割行的集合  
2.ORDER BY来对行进行排序  
3.feame语句定义子集   

feame语句的原理是cursor（游标）  

可以参考 https://www.cnblogs.com/xiongzaiqiren/p/sql-cursor.html

使用feame语句把其他行的数据拿到自己的行中  
LoadSample表  
| <ins>sample_date</ins> | load_val | 
| --- | --- |
| 2018-02-01 | 1024 |
| 2018-02-02 | 2366 |
| 2018-02-05 | 2366 |
| 2018-02-07 | 985 |
| 2018-02-08 | 780 |
| 2018-02-12 | 1000 |

例1，求最近的日期  
```
SELECT 
    sample_date AS cur_date,
    MIN(sample_date)
        OVER (
            ORDER BY sample_date ASC
            ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
        ) AS latest_date
FROM LoadSample;
```
功能说明  

SELECT 子句：  
	选择的列包括：  
		sample_date AS cur_date：当前行的日期，重命名为 cur_date。  
		MIN(sample_date) OVER (...) AS latest_date：  
			使用窗口函数 MIN，计算窗口范围内的最小值，并将结果命名为 latest_date。  
OVER 子句：  
	定义了窗口函数的计算范围。  
	ORDER BY sample_date ASC：  
		按 sample_date 列的升序排列数据。  
	ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING：  
		定义窗口范围为当前行的前一行。  
FROM LoadSample：  
	数据来源于 LoadSample 表。  


执行逻辑
按 sample_date 排序：  
	数据会按照 sample_date 的升序排列。  
定义窗口范围：  
	对于每一行，窗口范围仅包括当前行的前一行。  
计算窗口函数：  
	在窗口范围内，计算 sample_date 的最小值（实际上是前一行的值，因为窗口范围只有一行）。  

关键点总结  
窗口函数：  
	MIN 是一个窗口函数，用于计算窗口范围内的最小值。  
	在这种情况下，由于窗口范围只有一行，MIN 的结果就是该行的值。  
窗口范围：  
	ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING 将窗口范围限制为当前行的前一行。  
特殊情况：  
	如果当前行是第一行，则没有前一行，窗口为空，结果为 NULL。  
灵活性：  
	可以调整窗口范围，例如：  
	ROWS BETWEEN 2 PRECEDING AND 1 PRECEDING  
	这将计算当前行的前两行的最小值。  

接下来不仅要输出日期，也要输出出货量，有2种写法  
写法一：  
```sql
SELECT sample_date AS cur_date,
       load_val AS cur_load,
       MIN(sample_date)
          OVER (ORDER BY sample_date ASC
                ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) 
AS latest_date,
       MIN(load_val)
          OVER (ORDER BY sample_date ASC
                ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) 
AS latest_load
  FROM LoadSample;
```
通过相同的window重复使用。接下来，把它们整合在一起

```sql
SELECT sample_date AS cur_date,
       load_val    AS cur_load,
       MIN(sample_date) OVER W AS latest_date,
       MIN(load_val)    OVER W AS latest_load
  FROM LoadSample
 WINDOW W AS (ORDER BY sample_date ASC
              ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING);
```

Q1 feame可以不仅向前，也可以“向后”吗？

答：可以的。这个情况下使用FOLLOWING这个keyword。比如，最近的日期，往后一行。

```sql
SELECT sample_date AS cur_date,
       load_val    AS cur_load,
       MIN(sample_date) OVER W AS next_date,
       MIN(load_val)    OVER W AS next_load
  FROM LoadSample
 WINDOW W AS (ORDER BY sample_date ASC
              ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING);
```

输出结果

| cur_date   | cur_load | next_date  | next_load |
| ---------- | -------- | ---------- | --------- |
| 2018-02-01 | 1024     | 2018-02-02 | 2366      |
| 2018-02-02 | 2366     | 2018-02-05 | 2366      |
| 2018-02-05 | 2366     | 2018-02-07 | 985       |
| 2018-02-07 | 985      | 2018-02-08 | 780       |
| 2018-02-08 | 780      | 2018-02-12 | 1000      |
| 2018-02-12 | 1000     |            |           |

这次是把next_date和next_load列中，表示后面那一列数据。

还有，PRECEDING和FOLLOWING合并使用，表示“前后N行”的设置。



Q2 使用了MIN函数，这个是什么意思？

答：在这个例子中，feame（游标）的范围限定在1行的情况下，用MIN没有特别的意义。用MAX，AVG和SUM结果也是一样的。只有当feame的范围超过1行时，使用集约函数的本来意思会有作用。

```sql
-- これでも結果はMINと同じ
SELECT sample_date AS cur_date,
       load_val    AS cur_load,
       MAX(sample_date) OVER W AS latest_date,
       MAX(load_val)    OVER W AS latest_load
  FROM LoadSample
 WINDOW W AS (ORDER BY sample_date ASC
              ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING);
```



Q3 不是行而是基于“1天前”或“2天前”的列的数值进行feame设定，可以吗？

答：可以的。这个情况下用RANGE代替ROWS。

```sql
SELECT sample_date AS cur_date,
	load_val AS cur_load,
	MIN(sample_date)
		OVER (ORDER BY sample_date ASC
			RANGE BETWEEN interval '1' day PRECEDING
		  AND interval '1' day PRECEDING
			) AS day1_before,
		MIN(load_val)
			OVER (ORDER BY sample_date ASC
				RANGE BETWEEN interval '1' day PRECEDING
					AND interval '1' day PRECEDING
				) AS load_day1_before
FROM LoadSample;

```

输出结果

| cur_date | cur_load | day1_before | load_day1_before |
| -------- | -------- | ----------- | ---------------- |
| 18-02-01 | 1024     |             |                  |
| 18-02-02 | 2366     | 18-02-01    | 1024             |
| 18-02-05 | 2366     |             |                  |
| 18-02-07 | 985      |             |                  |
| 18-02-08 | 780      | 18-02-07    | 985              |
| 18-02-12 | 1000     |             |                  |

由于表格LoadSample的数据不是连续的，在1天前的数据不存在的情况下，day1_before和load_day1_before的列表示空值NULL。

最后总结feame语句如下

●ROWS：设定行的移动单位

●RANGE：设定列的移动单位。由ORDER BY指定的列作为基准

●n PRECENDING：向前移动n行（往小的方向移动）。n为正整数

●n FOLLOWING：向后移动n行（往大的方向移动）。n为正整数

●UNBOUNDED FOLLOWING：无限向上移动。从分区的第一行开始，进行累计计算。例如累计和、累计平均值等。

●UNBOUNDED FOLLOWING：无限向下移动。通常用于从当前行到分区末尾的计算，例如剩余总和、剩余平均值等。

●CURRENT ROW：现在行

行间比较的一般化

要求：扩大比较范围，从“最近的1天”到“再前1天”到“再前2天”。一直到“前n天”

```sql
SELECT sample_date AS cur_date,
	MIN(sample_date)
	  OVER (ORDER BY sample_date ASC
		ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
AS latest_1,
	MIN(sample_date)
	  OVER (ORDER BY sample_date ASC
		ROWS BETWEEN 2 PRECEDING AND 2 PRECEDING)
AS latest_2,
	MIN(sample_date)
	  OVER (ORDER BY sample_date ASC
		ROWS BETWEEN 3 PRECEDING AND 3 PRECEDING)
AS latest_3
 FROM LoadSample;
```

通过BETWEEN来指定“前1行”“前2行”“前3行”。可以通过这个方法进行扩展到n行。

window函数的实际是sort（种类，分类）

总结 まとめ

1.window函数的“window”，（原则上有顺序）意为“范围”

2.在window函数的构造中，PARTITION BY语句和ORDER BY语句虽然是有特征的行的集合，但是于通常使用简化的语法，反而难以意识到window窗口的存在。

3.PARTITON BY语句是GROUP BY从子句中减去聚合功能，只保留剪切功能ORDER BY子句对记录进行排序。

4.feame语句将光标的功能SQL的语法，可以定义以“当前记录”为中心的记录集合的范围

5.使用feame语句，可以把不同行的数据拿到同一行中，简化了行之间的比较。

6.关于window函数的内部运行，目前还是把行进行分类。未来有可能采用哈希hash。

练习问题

练习表格ServerLoadSample

| sever(サーバ) | sample_date(計測日) | load_val(負荷量) |
| ------------- | ------------------- | ---------------- |
| A             | 2018-02-01          | 1024             |
| A             | 2018-02-02          | 2366             |
| A             | 2018-02-05          | 2366             |
| A             | 2018-02-07          | 985              |
| A             | 2018-02-08          | 780              |
| A             | 2018-02-12          | 1000             |
| B             | 2018-02-01          | 54               |
| B             | 2018-02-02          | 39008            |
| B             | 2018-02-03          | 2900             |
| B             | 2018-02-04          | 556              |
| B             | 2018-02-05          | 12600            |
| B             | 2018-02-06          | 7309             |
| C             | 2018-02-01          | 1000             |
| C             | 2018-02-07          | 2000             |
| C             | 2018-02-16          | 500              |

1.预测输出结果

```sql
SELECT server, sample_date,
SUM(load_val) OVER () AS sum_load
 FROM ServerLoadSample;
```

结果如下表

| server | sample_date | sum_load |
| ------ | ----------- | -------- |
| A      | 2018-02-01  | 74448    |
| A      | 2018-02-02  | 74448    |
| A      | 2018-02-05  | 74448    |
| A      | 2018-02-07  | 74448    |
| A      | 2018-02-08  | 74448    |
| A      | 2018-02-12  | 74448    |
| B      | 2018-02-01  | 74448    |
| B      | 2018-02-02  | 74448    |
| B      | 2018-02-03  | 74448    |
| B      | 2018-02-04  | 74448    |
| B      | 2018-02-05  | 74448    |
| B      | 2018-02-06  | 74448    |
| C      | 2018-02-01  | 74448    |
| C      | 2018-02-07  | 74448    |
| C      | 2018-02-16  | 74448    |

sum_load这一列，都是74448。原因是，load_val下行的值全部求和。因为没有PARTITION BY语句，整个window被当成一个大隔板（partition）。这个和没有GROUP BY约束的集约函数的情况类似，整个表格作为1个大组进行处理。

还有，window函数中没有ORDER BY语句，没法对行进行排序进行累计计算。所以，sum_load的行就是整个求和的值。

2.预测输出结果

```sql
SELECT server, sample_date,
 SUM(load_val) OVER (PARTITION BY server) AS sum_load
 FROM ServerLoadSample;
```

这次加了PATITION BY 语句，想一下结果会有什么变化。

结果如下表

| server | sample_date | sum_load |
| ------ | ----------- | -------- |
| A      | 2018-02-01  | 8521     |
| A      | 2018-02-02  | 8521     |
| A      | 2018-02-05  | 8521     |
| A      | 2018-02-07  | 8521     |
| A      | 2018-02-08  | 8521     |
| A      | 2018-02-12  | 8521     |
| B      | 2018-02-01  | 62427    |
| B      | 2018-02-02  | 62427    |
| B      | 2018-02-03  | 62427    |
| B      | 2018-02-04  | 62427    |
| B      | 2018-02-05  | 62427    |
| B      | 2018-02-06  | 62427    |
| C      | 2018-02-01  | 3500     |
| C      | 2018-02-07  | 3500     |
| C      | 2018-02-16  | 3500     |

因为添加了PATITION BY语句，通过sever这一部分进行区分，然后运行SUM函数。可是依然没有ORDER BY，只是对区分部分进行约束。就成了计算sever的负担量的合计。



self join的使用方法

