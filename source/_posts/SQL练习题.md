---
title: SQL练习题
abbrlink: 780c7923
date: 2020-03-16 21:26:30
tags: 编程语言	
---

最近学了SQL基础，做点题练练手，

转载自：

-[经典SQL练习题(MySQL版)](https://blog.csdn.net/paul0127/article/details/82529216)

<!-- more -->

网上有一篇关于SQL的经典文章，[超经典SQL练习题，做完这些你的SQL就过关了](https://blog.csdn.net/flycat296/article/details/63681089)，引用和分析它的人很多，于是今天复习SQL的时候找来练了练手。原作者用的是SQL Server 2008，我在这里用的是MySQL 8.0.11（二者语法差别不大），文本编辑器用的是Atom 1.28.2（不知道大家用什么，反正用Atom写SQL确实丝质顺滑）。

笔者个人推荐DataGrip，可能是对jetbrain家的产品产生了依赖，还是觉得非常好用。

题目顺序和原文一致，但是我没有把所有题目都解一遍，因为很多题目是重复的。在每道题题目下我除了放SQL语句外，还把MySQL的运行输出结果放了上来，展示效果更直观一些。另外，因为数据量非常小，所以就没考虑SQL语句的性能优化，只求顺利完成题目，并尽可能写得简单些。

开始之前，先从[SQL常见的一些面试题(太有用啦)](https://www.cnblogs.com/diffrent/p/8854995.html)搬运几道我认为很不错的经典题目过来，这些题目的解法体现出来的方法和思路可以适用于本文的绝大部分题目，是必备的基础。

**1. 用一条SQL 语句 查询出每门课都大于80 分的学生姓名**

name   course grade
 张三    语文       81
 张三     数学       75
 李四     语文       76
 李四     数学       90
 王五     语文       81
 王五     数学       100
 王五     英语       90

```csharp
select name from table group by name having min(grade) > 80
```

**2. 现有学生表如下:**
 自动编号   学号   姓名 课程编号 课程名称 分数
 1        2005001 张三 0001     数学    69
 2        2005002 李四 0001      数学    89
 3        2005001 张三 0001      数学    69
 **删除除了自动编号不同, 其他都相同的学生冗余信息**



```csharp
delete tablename where 自动编号 not in (
    select min( 自动编号) 
    from tablename 
    group by 学号, 姓名, 课程编号, 课程名称, 分数
)
```

**3. 一个叫 team 的表，里面只有一个字段name, 一共有4 条纪录，分别是a,b,c,d, 对应四个球对，现在四个球对进行比赛，用一条sql 语句显示所有可能的比赛组合**



```css
select a.name, b.name
from team a, team b 
where a.name < b.name
```

**4. 请用SQL 语句实现：从TestDB 数据表中查询出所有月份的发生额都比101 科目相应月份的发生额高的科目。**
 请注意：TestDB 中有很多科目，都有1~12月份的发生额。
 AccID ：科目代码，Occmonth ：发生额月份，DebitOccur ：发生额。
 数据库名：JcyAudit ，数据集：Select * from TestDB



```csharp
select a.*
from TestDB a, 
    (select Occmonth, max(DebitOccur) as Debit101ccur 
    from TestDB 
    where AccID='101' 
    group by Occmonth) b
where a.Occmonth = b.Occmonth and a.DebitOccur > b.Debit101ccur
```

**5. 怎么把这样一个数据表**
 year   month amount
 1991   1     1.1
 1991   2     1.2
 1991   3     1.3
 1991   4     1.4
 1992   1     2.1
 1992   2     2.2
 1992   3     2.3
 1992   4     2.4
 **查成这样一个结果？**
 year m1   m2   m3   m4
 1991 1.1 1.2 1.3 1.4
 1992 2.1 2.2 2.3 2.4



```csharp
select year, 
    (select amount from table m where month=1 and m.year=table.year) as m1,
    (select amount from table m where month=2 and m.year=table.year) as m2,
    (select amount from table m where month=3 and m.year=table.year) as m3,
    (select amount from table m where month=4 and m.year=table.year) as m4
from table group by year
```

**6. 有表A，结构如下：**
 p_ID p_Num s_id
 1 10 01
 1 12 02
 2 8 01
 3 11 01
 3 8 03
 其中：p_ID为产品ID，p_Num为产品库存量，s_id为仓库ID。
 **请用SQL语句实现将上表中的数据合并，合并后的数据为：**
 p_ID s1_id s2_id s3_id
 1 10 12 0
 2 8 0 0
 3 11 0 8
 其中：s1_id为仓库1的库存量，s2_id为仓库2的库存量，s3_id为仓库3的库存量。如果该产品在某仓库中无库存量，那么就是0代替。

```csharp
select p_id,
    sum(case when s_id=1 then p_num else 0 end) as s1_id,
    sum(case when s_id=2 then p_num else 0 end) as s2_id,
    sum(case when s_id=3 then p_num else 0 end) as s3_id
from myPro group by p_id
```

------

下面进入正题。首先创建数据表：

**学生表 Student**

```csharp
create table Student(Sid varchar(6), Sname varchar(10), Sage datetime, Ssex varchar(10));
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('02' , '钱电' , '1990-12-21' , '男');
insert into Student values('03' , '孙风' , '1990-05-20' , '男');
insert into Student values('04' , '李云' , '1990-08-06' , '男');
insert into Student values('05' , '周梅' , '1991-12-01' , '女');
insert into Student values('06' , '吴兰' , '1992-03-01' , '女');
insert into Student values('07' , '郑竹' , '1989-07-01' , '女');
insert into Student values('08' , '王菊' , '1990-01-20' , '女')
```

**成绩表 SC**

```csharp
create table SC(Sid varchar(10), Cid varchar(10), score decimal(18,1));
insert into SC values('01' , '01' , 80);
insert into SC values('01' , '02' , 90);
insert into SC values('01' , '03' , 99);
insert into SC values('02' , '01' , 70);
insert into SC values('02' , '02' , 60);
insert into SC values('02' , '03' , 80);
insert into SC values('03' , '01' , 80);
insert into SC values('03' , '02' , 80);
insert into SC values('03' , '03' , 80);
insert into SC values('04' , '01' , 50);
insert into SC values('04' , '02' , 30);
insert into SC values('04' , '03' , 20);
insert into SC values('05' , '01' , 76);
insert into SC values('05' , '02' , 87);
insert into SC values('06' , '01' , 31);
insert into SC values('06' , '03' , 34);
insert into SC values('07' , '02' , 89);
insert into SC values('07' , '03' , 98)
```

**课程表 Course**

```csharp
create table Course(Cid varchar(10),Cname varchar(10),Tid varchar(10));
insert into Course values('01' , '语文' , '02');
insert into Course values('02' , '数学' , '01');
insert into Course values('03' , '英语' , '03')
```

**教师表 Teacher**

```csharp
create table Teacher(Tid varchar(10),Tname varchar(10));
insert into Teacher values('01' , '张三');
insert into Teacher values('02' , '李四');
insert into Teacher values('03' , '王五')
```

四张表之间的关联很简单：

![img](https:////upload-images.jianshu.io/upload_images/10870953-a9ce8600ab818b63.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

表格关联

（以下题目的顺序和原文相对应）

**1. 查询" 01 "课程比" 02 "课程成绩高的学生的信息及课程分数**

```csharp
select s.*, a.score as score_01, b.score as score_02
from student s,
     (select sid, score from sc where cid=01) a,
     (select sid, score from sc where cid=02) b
where a.sid = b.sid and a.score > b.score and s.sid = a.sid
```

```ruby
+------+--------+---------------------+------+----------+----------+
| Sid  | Sname  | Sage                | Ssex | score_01 | score_02 |
+------+--------+---------------------+------+----------+----------+
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |     70.0 |     60.0 |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |     50.0 |     30.0 |
+------+--------+---------------------+------+----------+----------+
2 rows in set (0.00 sec)
```

**2. 查询平均成绩大于等于 60 分的同学的学生编号和学生姓名和平均成绩**

```csharp
select s.sid, sname, avg(score) as avg_score
from student as s, sc
where s.sid = sc.sid
group by s.sid
having avg_score > 60
```

```ruby
+------+--------+-----------+
| sid  | sname  | avg_score |
+------+--------+-----------+
| 01   | 赵雷   |  89.66667 |
| 02   | 钱电   |  70.00000 |
| 03   | 孙风   |  80.00000 |
| 05   | 周梅   |  81.50000 |
| 07   | 郑竹   |  93.50000 |
+------+--------+-----------+
5 rows in set (0.00 sec)
```

**3. 查询在 SC 表存在成绩的学生信息**

```csharp
select * from student where sid in (select sid from sc where score is not null)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
| 05   | 周梅   | 1991-12-01 00:00:00 | 女   |
| 06   | 吴兰   | 1992-03-01 00:00:00 | 女   |
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |
+------+--------+---------------------+------+
7 rows in set (0.00 sec)
```

**4. 查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩(没成绩的显示为 null )**

这道题得用到left join或者right join，不能用where连接，因为题目说了要求有显示为null的，where是inner join，不会出现null，在这道题里会查不出第08号学生。

```csharp
select s.sid, s.sname, count(cid) as 选课总数, sum(score) as 总成绩
from student as s left join sc
on s.sid = sc.sid
group by s.sid
```

```ruby
+------+--------+--------------+-----------+
| sid  | sname  | 选课总数     | 总成绩    |
+------+--------+--------------+-----------+
| 01   | 赵雷   |            3 |     269.0 |
| 02   | 钱电   |            3 |     210.0 |
| 03   | 孙风   |            3 |     240.0 |
| 04   | 李云   |            3 |     100.0 |
| 05   | 周梅   |            2 |     163.0 |
| 06   | 吴兰   |            2 |      65.0 |
| 07   | 郑竹   |            2 |     187.0 |
| 08   | 王菊   |            0 |      NULL |
+------+--------+--------------+-----------+
8 rows in set (0.00 sec)
```

**4.1 查有成绩的学生信息**

```csharp
select s.sid, s.sname, count(*) as 选课总数, sum(score) as 总成绩,
    sum(case when cid = 01 then score else null end) as score_01,
    sum(case when cid = 02 then score else null end) as score_02,
    sum(case when cid = 03 then score else null end) as score_03
from student as s, sc
where s.sid = sc.sid
group by s.sid
```

```ruby
+------+--------+--------------+-----------+----------+----------+----------+
| sid  | sname  | 选课总数     | 总成绩    | score_01 | score_02 | score_03 |
+------+--------+--------------+-----------+----------+----------+----------+
| 01   | 赵雷   |            3 |     269.0 |     80.0 |     90.0 |     99.0 |
| 02   | 钱电   |            3 |     210.0 |     70.0 |     60.0 |     80.0 |
| 03   | 孙风   |            3 |     240.0 |     80.0 |     80.0 |     80.0 |
| 04   | 李云   |            3 |     100.0 |     50.0 |     30.0 |     20.0 |
| 05   | 周梅   |            2 |     163.0 |     76.0 |     87.0 |     NULL |
| 06   | 吴兰   |            2 |      65.0 |     31.0 |     NULL |     34.0 |
| 07   | 郑竹   |            2 |     187.0 |     NULL |     89.0 |     98.0 |
+------+--------+--------------+-----------+----------+----------+----------+
7 rows in set (0.00 sec)
```

**5. 查询「李」姓老师的数量**

```csharp
select count(tname) from teacher where tname like '李%'
```

```ruby
+--------------+
| count(tname) |
+--------------+
|            1 |
+--------------+
1 row in set (0.00 sec)
```

**6. 查询学过「张三」老师授课的同学的信息**

```csharp
select * from student where sid in (
    select sid from sc, course, teacher
    where sc.cid = course.cid
     and course.tid = teacher.tid
     and tname = '张三'
)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
| 05   | 周梅   | 1991-12-01 00:00:00 | 女   |
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |
+------+--------+---------------------+------+
6 rows in set (0.00 sec)
```

原作者的写法里面用到了等号 =，虽然得到同样的结果，但是这样写不太好，因为不确定张三老师是不是只教授一门课（只不过现在的数据量太小了而已），in 适用于一个或多个返回结果的情况，适应性比等号更广。

```csharp
select * from Student
where sid in(select distinct Sid from SC
where cid=(select Cid from Course
where Tid=(select Tid from Teacher where Tname='张三')))
```

**7. 查询没有学全所有课程的同学的信息**

```csharp
select * from student where sid in (select sid from sc group by sid having count(cid) < 3)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 05   | 周梅   | 1991-12-01 00:00:00 | 女   |
| 06   | 吴兰   | 1992-03-01 00:00:00 | 女   |
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |
+------+--------+---------------------+------+
3 rows in set (0.00 sec)
```

**9. 查询和" 01 "号的同学学习的课程完全相同的其他同学的信息**
 这道题号称是所有题目里最难的一道，我虽然做了出来，但是写法很麻烦，不必要。原作者写的很简洁：

```csharp
select * from Student
where Sid in(
    select Sid from SC
    where Cid in (select Cid from SC where Sid = '01') and Sid <>'01'
    group by Sid
    having COUNT(Cid)>=3
)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
+------+--------+---------------------+------+
3 rows in set (0.00 sec)
```

我写的就太麻烦啦。。

```csharp
select * from student where sid in (
    select B.sid
    from
        (select sid,
            sum(case when cid=01 then 1 else 0 end) as course_01,
            sum(case when cid=02 then 1 else 0 end) as course_02,
            sum(case when cid=03 then 1 else 0 end) as course_03
        from sc where sid = 01 group by sid) as A,
        (select sid,
            sum(case when cid=01 then 1 else 0 end) as course_01,
            sum(case when cid=02 then 1 else 0 end) as course_02,
            sum(case when cid=03 then 1 else 0 end) as course_03
        from sc where sid != 01 group by sid) as B
    where A.course_01=B.course_01 and A.course_02=B.course_02 and A.course_03=B.course_03
)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
+------+--------+---------------------+------+
3 rows in set (0.00 sec)
```

**8. 查询至少有一门课与学号为" 01 "的同学所学相同的同学的信息**

和第9题基本一致，还是原作者写的好一些

```csharp
select * from Student where Sid in(
    select distinct Sid from SC where Cid in(
        select Cid from SC where Sid='01'
    )
)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
| 05   | 周梅   | 1991-12-01 00:00:00 | 女   |
| 06   | 吴兰   | 1992-03-01 00:00:00 | 女   |
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |
+------+--------+---------------------+------+
7 rows in set (0.00 sec)
```

**10. 查询没学过"张三"老师讲授的任一门课程的学生姓名**

一般涉及到"任意"的都会用到not in这样的取反的结构：

```csharp
select sname from student
where sname not in (
    select s.sname
    from student as s, course as c, teacher as t, sc
    where s.sid = sc.sid
        and sc.cid = c.cid
        and c.tid = t.tid
        and t.tname = '张三'
)
```

```ruby
+--------+
| sname  |
+--------+
| 吴兰   |
| 王菊   |
+--------+
2 rows in set (0.00 sec)
```

**11. 查询两门及其以上不及格课程的同学的学号，姓名及其平均成绩**

```csharp
select s.sid, s.sname, avg(score)
from student as s, sc
where s.sid = sc.sid and score<60
group by s.sid
having count(score)>=2
```

```ruby
+------+--------+------------+
| sid  | sname  | avg(score) |
+------+--------+------------+
| 04   | 李云   |   33.33333 |
| 06   | 吴兰   |   32.50000 |
+------+--------+------------+
2 rows in set (0.00 sec)
```

**12. 检索" 01 "课程分数小于 60，按分数降序排列的学生信息**

```csharp
select s.* ,score
from student as s, sc
where cid = 01
  and score < 60
  and s.sid=sc.sid
order by score desc
```

```ruby
+------+--------+---------------------+------+-------+
| Sid  | Sname  | Sage                | Ssex | score |
+------+--------+---------------------+------+-------+
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |  50.0 |
| 06   | 吴兰   | 1992-03-01 00:00:00 | 女   |  31.0 |
+------+--------+---------------------+------+-------+
2 rows in set (0.00 sec)
```

**13. 按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩**

```csharp
select sid,
    sum(case when cid=01 then score else null end) as score_01,
    sum(case when cid=02 then score else null end) as score_02,
    sum(case when cid=03 then score else null end) as score_03,
    avg(score)
from sc group by sid
order by avg(score) desc
```

```ruby
+------+----------+----------+----------+------------+
| sid  | score_01 | score_02 | score_03 | avg(score) |
+------+----------+----------+----------+------------+
| 07   |     NULL |     89.0 |     98.0 |   93.50000 |
| 01   |     80.0 |     90.0 |     99.0 |   89.66667 |
| 05   |     76.0 |     87.0 |     NULL |   81.50000 |
| 03   |     80.0 |     80.0 |     80.0 |   80.00000 |
| 02   |     70.0 |     60.0 |     80.0 |   70.00000 |
| 04   |     50.0 |     30.0 |     20.0 |   33.33333 |
| 06   |     31.0 |     NULL |     34.0 |   32.50000 |
+------+----------+----------+----------+------------+
7 rows in set (0.00 sec)
```

**14. 查询各科成绩最高分、最低分和平均分，以如下形式显示：课程 ID，课程 name，最高分，最低分，平均分，及格率，中等率，优良率，优秀率(及格为>=60，中等为：70-80，优良为：80-90，优秀为：>=90）。
 要求输出课程号和选修人数，查询结果按人数降序排列，若人数相同，按课程号升序排列**

这道题熟练掌握case和sum的用法就没什么问题

```swift
select c.cid as 课程号, c.cname as 课程名称, count(*) as 选修人数,
    max(score) as 最高分, min(score) as 最低分, avg(score) as 平均分,
    sum(case when score >= 60 then 1 else 0 end)/count(*) as 及格率,
    sum(case when score >= 70 and score < 80 then 1 else 0 end)/count(*) as 中等率,
    sum(case when score >= 80 and score < 90 then 1 else 0 end)/count(*) as 优良率,
    sum(case when score >= 90 then 1 else 0 end)/count(*) as 优秀率
from sc, course c
where c.cid = sc.cid
group by c.cid
order by count(*) desc, c.cid asc
```

```ruby
+-----------+--------------+--------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| 课程号    | 课程名称      | 选修人数      | 最高分     | 最低分    | 平均分     | 及格率    | 中等率    | 优良率     | 优秀率     |
+-----------+--------------+--------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| 01        | 语文         |            6 |      80.0 |      31.0 |  64.50000 |    0.6667 |    0.3333 |    0.3333 |    0.0000 |
| 02        | 数学         |            6 |      90.0 |      30.0 |  72.66667 |    0.8333 |    0.0000 |    0.5000 |    0.1667 |
| 03        | 英语         |            6 |      99.0 |      20.0 |  68.50000 |    0.6667 |    0.0000 |    0.3333 |    0.3333 |
+-----------+--------------+--------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
3 rows in set (0.00 sec)
```

原作者的写法本质上和我是相同的，但是用了很多left join看起来有些冗余

```csharp
select distinct A.Cid,Cname,最高分,最低分,平均分,及格率,中等率,优良率,优秀率 from SC A
left join Course on A.Cid=Course.Cid
left join (select Cid,MAX(score)最高分,MIN(score)最低分,AVG(score)平均分 from SC group by Cid)B on A.Cid=B.Cid
left join (select Cid,(convert(decimal(5,2),(sum(case when score>=60 then 1 else 0 end)*1.00)/COUNT(*))*100)及格率 from SC group by Cid)C on A.Cid=C.Cid
left join (select Cid,(convert(decimal(5,2),(sum(case when score >=70 and score<80 then 1 else 0 end)*1.00)/COUNT(*))*100)中等率 from SC group by Cid)D on A.Cid=D.Cid
left join (select Cid,(convert(decimal(5,2),(sum(case when score >=80 and score<90 then 1 else 0 end)*1.00)/COUNT(*))*100)优良率 from SC group by Cid)E on A.Cid=E.Cid
left join (select Cid,(convert(decimal(5,2),(sum(case when score >=90 then 1 else 0 end)*1.00)/COUNT(*))*100)优秀率
from SC group by Cid)F on A.Cid=F.Cid
```

**15. 按平均成绩进行排序，显示总排名和各科排名，Score 重复时保留名次空缺**

原题目是按各科成绩进行排序，并显示排名， Score 重复时保留名次空缺。但是我没看明白什么意思，各科成绩如何排序？语文分数和数学分数有可比性吗？作者的写法是`select *,RANK()over(order by score desc)排名 from SC`，把所有的成绩都放到一块儿排序了，这没有意义，不可比。于是我修改了一下题目。

```csharp
select s.*, rank_01, rank_02, rank_03, rank_total
from student s
left join (select sid, rank() over(partition by cid order by score desc) as rank_01 from sc where cid=01) A on s.sid=A.sid
left join (select sid, rank() over(partition by cid order by score desc) as rank_02 from sc where cid=02) B on s.sid=B.sid
left join (select sid, rank() over(partition by cid order by score desc) as rank_03 from sc where cid=03) C on s.sid=C.sid
left join (select sid, rank() over(order by avg(score) desc) as rank_total from sc group by sid) D on s.sid=D.sid
order by rank_total asc
```

```ruby
+------+--------+---------------------+------+---------+---------+---------+------------+
| Sid  | Sname  | Sage                | Ssex | rank_01 | rank_02 | rank_03 | rank_total |
+------+--------+---------------------+------+---------+---------+---------+------------+
| 08   | 王菊   | 1990-01-20 00:00:00 | 女   |    NULL |    NULL |    NULL |       NULL |
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |    NULL |       2 |       2 |          1 |
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |       1 |       1 |       1 |          2 |
| 05   | 周梅   | 1991-12-01 00:00:00 | 女   |       3 |       3 |    NULL |          3 |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |       1 |       4 |       3 |          4 |
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |       4 |       5 |       3 |          5 |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |       5 |       6 |       6 |          6 |
| 06   | 吴兰   | 1992-03-01 00:00:00 | 女   |       6 |    NULL |       5 |          7 |
+------+--------+---------------------+------+---------+---------+---------+------------+
8 rows in set (0.00 sec)
```

**15.1 按平均成绩进行排序，显示总排名和各科排名，Score 重复时合并名次**

同样修改了一下题目。15题和15.1题的指向很明确了，就是rank()和dense_rank()的区别，也就是两个并列第一名之后的那个人是第三名(rank)还是第二名(dense_rank)的区别。

```csharp
select s.*, rank_01, rank_02, rank_03, rank_total
from student s
left join (select sid, dense_rank() over(partition by cid order by score desc) as rank_01 from sc where cid=01) A on s.sid=A.sid
left join (select sid, dense_rank() over(partition by cid order by score desc) as rank_02 from sc where cid=02) B on s.sid=B.sid
left join (select sid, dense_rank() over(partition by cid order by score desc) as rank_03 from sc where cid=03) C on s.sid=C.sid
left join (select sid, dense_rank() over(order by avg(score) desc) as rank_total from sc group by sid) D on s.sid=D.sid
order by rank_total asc
```

```ruby
+------+--------+---------------------+------+---------+---------+---------+------------+
| Sid  | Sname  | Sage                | Ssex | rank_01 | rank_02 | rank_03 | rank_total |
+------+--------+---------------------+------+---------+---------+---------+------------+
| 08   | 王菊   | 1990-01-20 00:00:00 | 女   |    NULL |    NULL |    NULL |       NULL |
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |    NULL |       2 |       2 |          1 |
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |       1 |       1 |       1 |          2 |
| 05   | 周梅   | 1991-12-01 00:00:00 | 女   |       2 |       3 |    NULL |          3 |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |       1 |       4 |       3 |          4 |
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |       3 |       5 |       3 |          5 |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |       4 |       6 |       5 |          6 |
| 06   | 吴兰   | 1992-03-01 00:00:00 | 女   |       5 |    NULL |       4 |          7 |
+------+--------+---------------------+------+---------+---------+---------+------------+
8 rows in set (0.00 sec)
```

**17. 统计各科成绩各分数段人数：课程编号，课程名称，[100-85]，[85-70]，[70-60]，[60-0] 及所占百分比**

```csharp
select c.cid as 课程编号, c.cname as 课程名称, A.*
from course as c,
(select cid,
    sum(case when score >= 85 then 1 else 0 end)/count(*) as 100_85,
    sum(case when score >= 70 and score < 85 then 1 else 0 end)/count(*) as 85_70,
    sum(case when score >= 60 and score < 70 then 1 else 0 end)/count(*) as 70_60,
    sum(case when score < 60 then 1 else 0 end)/count(*) as 60_0
from sc group by cid) as A
where c.cid = A.cid
```

```ruby
+--------------+--------------+------+--------+--------+--------+--------+
| 课程编号     | 课程名称      | cid  | 100_85 | 85_70  | 70_60  | 60_0   |
+--------------+--------------+------+--------+--------+--------+--------+
| 01           | 语文         | 01   | 0.0000 | 0.6667 | 0.0000 | 0.3333 |
| 02           | 数学         | 02   | 0.5000 | 0.1667 | 0.1667 | 0.1667 |
| 03           | 英语         | 03   | 0.3333 | 0.3333 | 0.0000 | 0.3333 |
+--------------+--------------+------+--------+--------+--------+--------+
3 rows in set (0.00 sec)
```

**18. 查询各科成绩前三名的记录**

这是我比较喜欢的一道题目，非常经典。

```csharp
select * from (select *, rank() over(partition by cid order by score desc) as graderank from sc) A 
where A.graderank <= 3
```

```ruby
| Sid  | Cid  | score | graderank |
+------+------+-------+-----------+
| 01   | 01   |  80.0 |         1 |
| 03   | 01   |  80.0 |         1 |
| 05   | 01   |  76.0 |         3 |
| 01   | 02   |  90.0 |         1 |
| 07   | 02   |  89.0 |         2 |
| 05   | 02   |  87.0 |         3 |
| 01   | 03   |  99.0 |         1 |
| 07   | 03   |  98.0 |         2 |
| 02   | 03   |  80.0 |         3 |
| 03   | 03   |  80.0 |         3 |
+------+------+-------+-----------+
10 rows in set (0.00 sec)
```

**20. 查询出只选修两门课程的学生学号和姓名**

```csharp
select s.sid, s.sname, count(cid)
from student s, sc
where s.sid = sc.sid
group by s.sid
having count(cid)=2
```

```ruby
+------+--------+------------+
| sid  | sname  | count(cid) |
+------+--------+------------+
| 05   | 周梅   |          2 |
| 06   | 吴兰   |          2 |
| 07   | 郑竹   |          2 |
+------+--------+------------+
3 rows in set (0.00 sec)
```

**22. 查询名字中含有「风」字的学生信息**

```csharp
select * from student where sname like '%风%'
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
+------+--------+---------------------+------+
1 row in set (0.00 sec)
```

**24. 查询 1990 年出生的学生名单**

```csharp
select * from student where year(sage) = 1990
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |
| 02   | 钱电   | 1990-12-21 00:00:00 | 男   |
| 03   | 孙风   | 1990-05-20 00:00:00 | 男   |
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
| 08   | 王菊   | 1990-01-20 00:00:00 | 女   |
+------+--------+---------------------+------+
5 rows in set (0.00 sec)
```

**33. 成绩不重复，查询选修「张三」老师所授课程的学生中，成绩最高的学生信息及其成绩**

```swift
select s.*, max(score)
from student s, teacher t, course c, sc
where s.sid = sc.sid
    and sc.cid = c.cid
    and c.tid = t.tid
    and t.tname = '张三'
```

```ruby
+------+--------+---------------------+------+------------+
| Sid  | Sname  | Sage                | Ssex | max(score) |
+------+--------+---------------------+------+------------+
| 01   | 赵雷   | 1990-01-01 00:00:00 | 男   |       90.0 |
+------+--------+---------------------+------+------------+
1 row in set (0.00 sec)
```

**34. 成绩有重复的情况下，查询选修「张三」老师所授课程的学生中，成绩最高的学生信息及其成绩**



```csharp
select * from (
    select *, DENSE_RANK() over (order by score desc) A
    from SC
    where Cid = (select Cid from Course where Tid=(select Tid from Teacher where Tname='张三'))
) B
where B.A=1
```

```ruby
+------+------+-------+---+
| Sid  | Cid  | score | A |
+------+------+-------+---+
| 01   | 02   |  90.0 | 1 |
+------+------+-------+---+
1 row in set (0.00 sec)
```

**40. 查询各学生的年龄，只按年份来算**



```csharp
select sname, year(now())-year(sage) as age from student
```

```ruby
+--------+------+
| sname  | age  |
+--------+------+
| 赵雷   |   28 |
| 钱电   |   28 |
| 孙风   |   28 |
| 李云   |   28 |
| 周梅   |   27 |
| 吴兰   |   26 |
| 郑竹   |   29 |
| 王菊   |   28 |
+--------+------+
8 rows in set (0.00 sec)
```

**41. 按照出生日期来算，当前月日 < 出生年月的月日则，年龄减一**

```csharp
select sname, timestampdiff(year, sage, now()) as age from student
```

```ruby
+--------+------+
| sname  | age  |
+--------+------+
| 赵雷   |   28 |
| 钱电   |   27 |
| 孙风   |   28 |
| 李云   |   27 |
| 周梅   |   26 |
| 吴兰   |   26 |
| 郑竹   |   29 |
| 王菊   |   28 |
+--------+------+
8 rows in set (0.00 sec)  
```

**42. 查询本周过生日的学生**



```csharp
select * from student where week(now()) = week(sage)
```



```bash
Empty set (0.00 sec)
```

**43. 查询下周过生日的学生**



```csharp
select * from student where (week(now())+1) = week(sage)
```



```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
+------+--------+---------------------+------+
1 row in set (0.00 sec)
```

**44. 查询本月过生日的学生**



```csharp
select * from student where month(now()) = month(sage)
```



```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 07   | 郑竹   | 1989-07-01 00:00:00 | 女   |
+------+--------+---------------------+------+
1 row in set (0.00 sec)
```

**45. 查询下月过生日的学生**

```csharp
select * from student where (month(now())+1) = month(sage)
```

```ruby
+------+--------+---------------------+------+
| Sid  | Sname  | Sage                | Ssex |
+------+--------+---------------------+------+
| 04   | 李云   | 1990-08-06 00:00:00 | 男   |
+------+--------+---------------------+------+
1 row in set (0.00 sec)
```