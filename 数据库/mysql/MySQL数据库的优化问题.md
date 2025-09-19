## MySQL数据库的优化问题

### Mysql 的语句分为哪几类

- 数据据定义语言DDL（Data Definition Language）：主要有CREATE，DROP，ALTER等对逻辑结构有操作的，包括表结构、视图和索引。

- 数据库查询语言DQL（Data Query Language）：主要以SELECT为主

- 数据操纵语言DML（Data Manipulation Language）：主要包括INSERT，UPDATE，DELETE

- 数据控制功能DCL（Data Control Language）：主要是权限控制能操作，包括GRANT，REVOKE，COMMIT，ROLLBACK等。

### SQL 约束有哪些

- 主键约束：主键为在表中存在一列或者多列的组合，能唯一标识表中的每一行。一个表只有一个主键，并且主键约束的列不能为空。

- 外键约束：外键约束是指用于在两个表之间建立关系，需要指定引用主表的哪一列。只有主表的主键可以被从表用作外键，被约束的从表的列可以不是主键，所以创建外键约束需要先定义主表的主键，然后定义从表的外键。

- 唯一约束：确保表中的一列数据没有相同的值，一个表可以定义多个唯一约束。

- 默认约束：在插入新数据时，如果该行没有指定数据，系统将默认值赋给该行，如果没有设置没默认值，则为NULL。

- Check约束：Check会通过逻辑表达式来判断数据的有效性，用来限制输入一列或者多列的值的范围。在列更新数据时，输入的内容必须满足Check约束的条件。

### 什么是子查询

子查询：把一个查询结果给另一个查询使用。

子查询可以分为几类

- 标量子查询：指子查询返回的是一个值，可以使用 =,>,<,>=,<=,<>等操作符对子查询标量结果进行比较，一般子查询会放在比较式的右侧。

  ```sql
  SELECT * FROM user WHERE age = (SELECT max(age) from user)  //查询年纪最大的人
  ```

- 列子查询：指子查询的结果是n行一列，一般应用于对表的某个字段进行查询返回。可以使用IN、ANY、SOME和ALL等操作符，不能直接使用

  ```sql
  SELECT num1 FROM table1 WHERE num1 > ANY (SELECT num2 FROM table2)
  ```

