---
title: MySql杂谈
date: 2016-11-19 11:06:51
tags:
  - MySQL
---
MySQL 闲杂知识点

## 1. MySQL concat

&emsp;&emsp;我们使用MySQL中的concat链接字符串函数的时候，如果链接的字段中有一个NULL值的话，我们最后出来的结果就是NULL，显然这不是我们想要的。我们可以使用coalesce函数来对NULL值初始化。

```MySQL
SELECT `atfcapi`.`tb_api_trace`.`git_version`,
       concat('Request: ',coalesce(`atfcapi`.`tb_api_trace`.`parameters_change_list`,''),'\nResponse: ',coalesce(`atfcapi`.`tb_api_trace`.`responses_change_list`,'')) AS `change`
FROM `atfcapi`.`tb_api_trace`
WHERE `atfcapi`.`tb_api_trace`.`tb_pathid` = 4217
ORDER BY `atfcapi`.`tb_api_trace`.`updated_at` DESC
```
具体可以参考 http://stackoverflow.com/questions/8530632/concat-values-in-mysql-queryto-handle-null-values

## 2. MySQL 查询当天数据

```MySQL
SELECT *
FROM TABLE
WHERE date(TIME) = curdate();
```
```MySQL
SELECT *
FROM TABLE
WHERE TO_DAYS(TIME) = TO_DAYS(NOW())
```
## 3. MySQL count(\*) count(1) count([列名])

&emsp;&emsp;Count(1)和Count(\*)实际上的意思是，评估Count（）中的表达式是否为NULL，如果为NULL则不计数，而非NULL则会计数。比如我们看如下代码所示，在Count中指定NULL（优化器不允许显式指定NULL，因此需要赋值给变量才能指定）。
```MySQL
DECLARE @x3x INT
SET @x3x=NULL
SELECT COUNT(@x3x) FROM TableName
```
因此当你指定Count(\*） 或者Count（1）或者无论Count('anything')时结果都会一样，因为这些值都不为NULL。对于Count（列）来说，同样适用于上面规则，评估列中每一行的值是否为NULL，如果为NULL则不计数，不为NULL则计数。因此Count（列）会计算列或这列的组合不为空的计数。
