title: 数据库递归查询：MySQL VS Sequelize
date: 2019-04-12 09:49:21
tags:
---
[一、前言](#一、前言) 
[二、MySQL实现](#二、MySQL实现) 
[三、Sequqlize实现](#三、Sequqlize实现) 
[四、总结](#四、总结)
[参考文档](#参考文档)

## 一、前言

最近在做团队的排期系统改版时涉及到数据库的递归查询问题，有一个需求数据表，表中的需求数据以parentId为外键定义数据的继承关系，需求之间的关系呈现树状关系。需求数据表如下：

```
mysql> desc needs;
+----------+-------------+------+-----+---------+----------------+
| Field    | Type        | Null | Key | Default | Extra          |
+----------+-------------+------+-----+---------+----------------+
| id       | int(11)     | NO   | PRI | NULL    | auto_increment |
| name     | varchar(45) | YES  |     | NULL    |                |
| parentId | int(11)     | YES  |     | NULL    |                |
+----------+-------------+------+-----+---------+----------------+
3 rows in set (0.01 sec)
```

目前有这样的需求需要根据某个根需求，查找出全部的层级的子需求。

例如A需求的树状结构如下：

![2019-04-12-数据库递归查询](MySQL-VS-Sequelize/needTree.jpg)

数据如下：
```
mysql> select * from needs;
+----+------+----------+
| id | name | parentId |
+----+------+----------+
|  1 | A    |     NULL |
|  2 | B    |        1 |
|  3 | C    |        1 |
|  4 | D    |        2 |
|  5 | E    |        2 |
|  6 | F    |        3 |
|  7 | G    |        3 |
|  8 | H    |        5 |
|  9 | I    |        5 |
| 10 | J    |        8 |
+----+------+----------+
10 rows in set (0.00 sec)
```

<br/>

## 二、MySQL实现

### 1.自定义函数实现

实现思路：首先根据子级`parenId`等于父级`id`的关系循环找出所有的层级关系数据的id，再拉出所有这些id的数据。

**（1）函数声明**

```
DELIMITER // 
CREATE FUNCTION `getParentList`(rootId INT)
    RETURNS char(400)
    BEGIN
      DECLARE fid int default 1;
      DECLARE str char(44) default rootId;
      WHILE rootId > 0 DO
      SET fid=(SELECT parentId FROM needs WHERE id=rootId);
     IF fid > 0 THEN
     SET str=CONCAT(str , ',' , fid);
     SET rootId=fid;
     ELSE SET rootId=fid;
     END IF;
     END WHILE;
  return  str;
  END //
```

语法解释：

`DELIMITER`：定义MySQL的分隔符为`//`，默认分隔符是`;`，为了防止函数内使用`;`中断函数

`CREATE FUNCTION 函数名(参数) RETURNS 返回值类型`：自定义函数

`DECLARE`：声明变量

`WHILE 条件 DO 循环体`：while循环

`IF 条件 THEN 内容体 ELSE 内容体`：if判断

`SET 变量=值`：存储值

`CONCAT(str1,str2,...)`：函数，用于将多个字符串连接成一个字符串

<br/>

**（2）函数调用**

```
mysql> DELIMITER;
    -> SELECT getNeedChild(1);
+-----------------------+
| getNeedChild(1)       |
+-----------------------+
| ,1,2,3,4,5,6,7,8,9,10 |
+-----------------------+
1 row in set (0.01 sec)
```

语法解释：

`DELIMITER;`：由于之前执行了`DELIMITER //`修改了分隔符，因此需要重新调用修改分隔符为`;`

`SELECT 函数()`：调用函数并搜索出结果

<br/>

**（3）结合FIND_IN_SET，拉取出所有的子需求**

```
mysql> SELECT * FROM needs WHERE FIND_IN_SET(ID , getNeedChild(1));
+----+------+----------+
| id | name | parentId |
+----+------+----------+
|  1 | A    |     NULL |
|  2 | B    |        1 |
|  3 | C    |        1 |
|  4 | D    |        2 |
|  5 | E    |        2 |
|  6 | F    |        3 |
|  7 | G    |        3 |
|  8 | H    |        5 |
|  9 | I    |        5 |
| 10 | J    |        8 |
+----+------+----------+
10 rows in set (0.03 sec)
```

`FIND_IN_SET(str,strlist)`：函数，查询字段`strlist`中包含`str`的结果，`strlist`中以`,`分割各项

<br/>

### 2.递归CTE实现

**（1）递归CTE介绍**

CTE（common table expression）为公共表表达式，可以对定义的表达式进行自引用查询。在MySQL 8.0版以上才支持。

递归CTE由三个部分组成：初始查询部分、递归查询部分、终止递归条件。

语法如下：

```
WITH RECURSIVE cte_name AS(
    initial_query       -- 初始查询部分
    UNION ALL           -- 递归查询与初始查询部分连接查询
    recursive_query     -- 递归查询部分
)
SELECT * FROM cte_name
```

更多CTE介绍可以查看文档：[A Definitive Guide To MySQL Recursive CTE](http://www.mysqltutorial.org/mysql-recursive-cte/)

<br/>

**（2）递归CTE实现**

```
WITH RECURSIVE needsTree AS
( SELECT id,
         name,
         parentId,
         1 lvl
    FROM needs
    WHERE id = 1 
  UNION ALL
  SELECT nd.id,
         nd.name,
         nd.parentId,
         lvl+1
    FROM needs AS nd
    JOIN needsTree AS nt ON nt.id = nd.parentId 
)
SELECT * FROM needsTree ORDER BY lvl;
```

实现解释：

初始查询部分：找出一级需求

递归查询部分：找出子级需求

终止递归条件：子级的`parentId`等于父级的`id`

查询结果：

```
+------+------+----------+------+
| id   | name | parentId | lvl  |
+------+------+----------+------+
|    1 | A    | NULL     |    1 |
|    2 | B    | 1        |    2 |
|    3 | C    | 1        |    2 |
|    6 | F    | 3        |    3 |
|    7 | G    | 3        |    3 |
|    4 | D    | 2        |    3 |
|    5 | E    | 2        |    3 |
|    8 | H    | 5        |    4 |
|    9 | I    | 5        |    4 |
|   10 | J    | 8        |    5 |
+------+------+----------+------+
10 rows in set (0.00 sec)
```

## 三、Sequqlize实现

### 1.Sequelize介绍

Sequelize是Node.js的ORM框架，能够把关系数据库的表结构映射到对象上，支持数据库Postgres、MySQL、 MariaDB、 SQLite and Microsoft SQL Server。在这次的排期系统后台开发中，我选择了该框架来操作数据库，可以更方便地处理数据。

更多Sequelize介绍可以查看官方文档：[Sequelize官方文档](http://docs.sequelizejs.com/)。

### 2.递归实现

**1.连接mysql数据库**

```
var Sequelize = require('sequelize');

const sequelize = new Sequelize('schedule' , 'root' , '12345678' , {
    host : '127.0.0.1',
    dialect : 'mysql',
    port : '3306',
})

module.exports = {
    sequelize
}
```

语法解释：

`new Sequelize(databse , username , password , options)`：实例化Sequelize，连接数据库

```
options = {
    host,   //数据库主机
    dialect,    //数据库
    port        //数据库端口号，默认为3306
}
```

<br/>

**2.定义数据表的schema模型表**

```
module.exports = function(sequelize, DataTypes) {
  return sequelize.define('needs', {
    id: {
      type: DataTypes.INTEGER(11),
      allowNull: false,
      primaryKey: true,
      autoIncrement: true
    },
    name: {
      type: DataTypes.STRING(45),
      allowNull: false
    },
    parentId: {
      type: DataTypes.INTEGER(11),
      allowNull: true,
    },
  }, {
    tableName: 'needs',
    timestamps: false
  });
};
```

语法解释：

`sequelize.define(modelName , attribute , options)`：定义数据表的模型，相当于定义数据表。

`attribute`：一个对象，为数据表对应的列项，`key`值为对应的列项名，`value`为对应列项的定义，比如数据类型、是否主键、是否必需等

`options`：数据表的一些配置。比如对应的数据表名`tableName`、是否需要时间戳`timestamp`等

<br/>

3.导入数据表模型

```
const { sequelize } = require('../config/db');

// 导入数据表模型
const Needs = sequelize.import('./needs.js');
```

语法解释：

`sequelize.import(path)`：导入数据表模型

<br/>

4.递归查询

实现思路：跟CTE实现思路相似，先找出找出一级需求，再递归找出子需求。

```
class NeedModule{
    constructor(id){
        this.id = id;
    }
    async getNeedsTree(){
        let rootNeeds = await Needs.findAll({
            where : { 
                id : this.id 
            }
        })
        rootNeeds = await this.getChildNeeds(rootNeeds);
        return rootNeeds;
    }
    async getChildNeeds(rootNeeds){
        let expendPromise = [];
        rootNeeds.forEach(item => {
            expendPromise.push(Needs.findAll({
                where : {
                    parentId : item.id
                }
            }))
        })
        let child = await Promise.all(expendPromise);
        for(let [idx , item] of child.entries()){
            if(item.length > 0){
                item = await getChildNeeds(item);
            }
            rootNeeds[idx].child = item;
        }
        return rootNeeds;
    }
}
```

语法解释：

`findALL(options)`：查询多条数据

`options`：查询配置

- `options.where`：查询条件

查询结果如下：

![2019-04-12-数据库递归查询](MySQL-VS-Sequelize/sequelizeResult.jpg)

从搜索结果可以看出，使用Sequelize查询可以更好的给层级数据划分层级存储。

<br/>

### 3.nested属性实现

Sequelize的`findAll`方法中的`nested`属性可以根据连接关系找出继承关系的数据。

**1.定义表关系**

由于需要需求表进行自连接查询，因此需要先定义表关系。需求表自身关系以父需求为主查询是一对多关系，因此使用`hasMany`定义关系。

```
Needs.hasMany(
    Needs, 
    {
        as: 'child', 
        foreignKey: 'parentId'
    }
);
```

语法解释：

`sourceModel.hasMany(targetModel, options)`：定义源模型和目标模型的表是一对多关系，外键会添加到目标模型中

`options`：定义表关系的一些属性。如`as`定义连接查询时，目标模型的别名。`foreignKey`为外键名。

**2.自连接查询**

```
async getNeedTree(id){
    return await Needs.findAll({
        where : {
            id 
        },
        include : {
            model: Needs,
            as:'child', 
            required : false,
            include : {
                all : true,
                nested : true,
            }
        }
    })
}
```

语法解释：

`include`：连接查询列表

- `include.model`：连接查询的模型

- `include.as`：连接查询模型的别名

- `include.requeired`：如果为`true`，连接查询为内连接。`false`为左连接。如果有`where`默认为`true`，其他情况默认为`false`。

- `include.all`：嵌套查询所有的模型

- `include.nested`：嵌套查询

使用此方法，查询最深的子级结果为三层。如果能保证数据继承关系最深为三层，可以使用此方法。

## 四、总结

在MySQL 8+可以使用CTE实现，相对于自定义函数实现可以使用更少的代码量实现，且使用`WITH...AS`可以优化递归查询。Sequelize目前支持CTE，但仅支持PostgreSQL、SQLite、MSSQL数据库，如果有更好的实现方式，可以分享下哦(≧▽≦)

## 参考文档

[1.mysql 递归查询 http://www.cnblogs.com/xiaoxi/p/5942805.html](http://www.cnblogs.com/xiaoxi/p/5942805.html)

[2.Managing Hierarchical Data in MySQL Using the Adjacency List Model](http://www.mysqltutorial.org/mysql-adjacency-list-tree/)

[3.A Definitive Guide To MySQL Recursive CTE](http://www.mysqltutorial.org/mysql-recursive-cte/)

[4.http://docs.sequelizejs.com/](http://docs.sequelizejs.com/)
