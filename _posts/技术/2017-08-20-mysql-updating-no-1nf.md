---
layout: post
title: 不满足第一范式的MySQL数据库更新操作-备忘
category: 技术
tags: MySQL
keywords: MySQL
description: 
---


#### 首先简单回顾下数据库的三大范式

[数据库设计三大范式](http://www.cnblogs.com/linjiqin/archive/2012/04/01/2428695.html)
> 1．第一范式(确保每列保持原子性)
> 
> 第一范式是最基本的范式。如果数据库表中的所有字段值都是不可分解的原子值，就说明该数据库表满足了第一范式。
> 
> 2．第二范式(确保表中的每列都和主键相关)
> 
> 第二范式在第一范式的基础之上更进一层。第二范式需要确保数据库表中的每一列都和主键相关，而不能只与主键的某一部分相关（主要针对联合主键而言）。也就是说在一个数据库表中，一个表中只能保存一种数据，不可以把多种数据保存在同一张数据库表中。
> 
> 3．第三范式(确保每列都和主键列直接相关,而不是间接相关)
> 
> 第三范式需要确保数据表中的每一列数据都和主键直接相关，而不能间接相关。

---

#### 实际情况

虽然设计数据库表结构的时候，不建议将多个值写在一个字段里面，但在实际情况中却有可能因为各种原因而这样做。

OA中进出操作间申请表的数据库设计不符合第一范式的要求，具体表现为：所有进出操作间的开发人员存在同一个字段内，彼此用“,”隔开；进入操作间的业务人员也存在同一个字段内。

***进出操作间申请表***
apply_id  | dev_peoples 
---|---
 1 | 张三,李四
 2 | 李四,王五,小二

***用户表***
id | name
---|---
1 | 小二
2 | 张三
3 | 李四
4 | 王五


---

#### 问题

在后面的开发过程中，想把**进出操作间申请表**中开发人员字段存的人员姓名改为id，这时候就遇到了问题。

如果我当时的设计是这样的，多一张表来记录申请中的人员信息：

***进出操作间申请人员信息表***
apply_id | dev_people
---|---
1 | 张三
1 | 李四
2 | 李四
2 | 王五
2 | 小二

每行记录的**dev_people**都只存了一个值，那在处理问题的时候就很简单，只需要关联***用户表***进行update操作即可，类似以下操作：

```
update 进出操作间申请人员信息表 apply
join 用户表 u on apply.dev_people = u.name
set apply.dev_people = u.id
```

但是，在不改变表结构的情况下，如何将“张三,李四”更新为“2,3”呢？

---

#### 思考

##### 1. 步骤一

第一反应就是用MySQL自带的replace方法，

```
update 进出操作间申请表
set dev_peoples = replace(dev_peoples,'李四','3')
```
然后就能得到以下结果：
***进出操作间申请表***
apply_id  | dev_peoples 
---|---
 1 | 张三,3
 2 | 3,王五,小二
 
 那么要如何循环执行n次replace呢？（很明显，n为***用户表***的行数）
 当时跟庆峰和智锋讨论的时候，庆峰想了个直接手动写出n条update语句的方法来做，但我和智锋觉得，应该有更加“高级”的做法，而不需要用这种靠体力劳动实现的方式。
 
 ##### 2. 步骤二
 
 在查找资料的时候，发现一篇博客：
 [MySQL逗号分割字段的行列转换技巧](http://www.cnblogs.com/cenalulu/archive/2012/08/20/2647463.html)，利用里面的方法可以得到中间表***进出操作间申请人员信息表***。
 
```
select a.id, 
substring_index(a.dev_peoples,',',b.help_topic_id+1),substring_index(substring_index(a.dev_peoples,',',b.help_topic_id+1),',',-1)  as substring   
from 进出操作间申请表 a join mysql.help_topic b 
on b.help_topic_id < (length(a.dev_peoples)-length(replace(a.dev_peoples,',',''))+1) 
order by a.id;
```
在on条件中<右边的条件得到的是dev_peoples字段按‘，’分割后有多少个值。而两层substring_index方法得到的分别是：取dev_peoples字段从1到i（i<=n)段‘，’分割开的值，和i段中的最后一个‘，’分开的值。

得到以下结果：

a.id | substring
---|---
1 | 张三
1 | 李四
2 | 李四
2 | 王五
2 | 小二

然后便可以关联用户表,得到用户id，

```
select tmp.id, u.id from 用户表 u join
(select a.id, 
substring_index(a.dev_peoples,',',b.help_topic_id+1),substring_index(substring_index(a.dev_peoples,',',b.help_topic_id+1),',',-1)  as substring   
from 进出操作间申请表 a join mysql.help_topic b 
on b.help_topic_id < (length(a.dev_peoples)-length(replace(a.dev_peoples,',',''))+1) 
order by a.id,b.help_topic_id) tmp 
on u.name = tmp.substring
```
得到以下结果：

tmp.id | u.id
---|---
1 | 2
1 | 3
2 | 3
2 | 4
2 | 1

下一步的问题就是：如何将分开的id再恢复成期望的"2,3"和"3,4,1"形式呢？

##### 3. 步骤三

在查找资料的时候，还发现了另一篇博客：[批量更新逗号隔开的名称 （部门里面将多个用逗号隔开的ID转换成用逗号隔开的名称）（mysql）](http://blog.csdn.net/sunshine901106/article/details/48051675)一开始的时候还看不懂里面的SQL语句，后来直接依样画葫芦地把表名和字段替换成自己的，就莫名地成功了，再仔细看了下SQL语句，发现一个叫**group_concat**的方法。继续搜索这个关键字后发现，这就是我所需要的方法。（具体用法见[MySQL GROUP_CONCAT Function](http://www.mysqltutorial.org/mysql-group_concat/)）

对步骤二的SQL语句进行修改：

```
-- 修改
select tmp.id, group_concat(u.id) from 用户表 u join
(select a.id, 
substring_index(a.dev_peoples,',',b.help_topic_id+1),substring_index(substring_index(a.dev_peoples,',',b.help_topic_id+1),',',-1)  as substring   
from 进出操作间申请表 a join mysql.help_topic b 
on b.help_topic_id < (length(a.dev_peoples)-length(replace(a.dev_peoples,',',''))+1) 
order by a.id,b.help_topic_id) tmp 
on u.name = tmp.substring
-- 增加
groupby tmp.id
```

得到：
tmp.id | u.id
---|---
1 | 2,3
2 | 3,4,1

##### 4. 步骤四

最后一步就是步骤三得到的临时结果与***进出操作间申请表***关联，然后进行update操作即可，关联条件为id。

---

#### 总结

通过这个问题的解决过程，了解了goup_concat和mysql自带的help_topic表等之前不知道的知识。
对高手而已，也许这个问题不算什么问题，所以在数据库的使用方面还需要多学习，最后，如果大家知道其他更简单的操作（比如利用数据存储过程可以更加简单地实现？），也希望能告诉我，非常感谢！
