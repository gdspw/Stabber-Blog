---
layout: post
title:  "新增mysql用户，并限制访问数据库"
categories: it
tags:  [Mysql,IT]
excerpt: 记录如何在mysql新增用户，并设置访问的数据的权限...
---

代码如下：

```mysql
CREATE USER 'yourName'@'%' IDENTIFIED BY 'yourPassWord';

GRANT SELECT, INSERT, UPDATE, REFERENCES, DELETE, CREATE, DROP, ALTER, INDEX, TRIGGER, CREATE VIEW, SHOW VIEW, EXECUTE, ALTER ROUTINE, CREATE ROUTINE, CREATE TEMPORARY TABLES, LOCK TABLES, EVENT ON `yourDbName`.* TO 'yourName'@'%';

GRANT GRANT OPTION ON `yourDbName`.* TO 'yourName'@'%';
```

