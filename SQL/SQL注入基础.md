### 系统库释意:
![[Pasted image 20250123170828.png]]
重要的四个库，前两个更关键
**information_schema** 库：是信息数据库，其中保存着**关于MySQL服务器所维护的所有其他数据库的信息**，比如数据库名，数据库表，表字段的数据类型与访问权限等。Web渗透过程中用途很大。
*查库：`SHOW TABLES FROM information_schema;`
其中比较重要的表有：
SCHEMATA*表：提供了当前MySQL实例中所有的数据库信息，`show databases`结果取之此表
*TABLES*表：提供了关于数据中表的信息，如**表名**
*COLUMNS*表：提供了表中的列信息，详细描述了某张表的所有列已经每个列的信息，如**字段**

**mysql**库：MySQL的核心数据库，主要负责存储数据库的用户、权限设置、 关键字等mysql自己需要使用的控制和管理信息。
查用户：`select user host from mysql.user;`

**performance_schema**库：内存数据库，数据放在内存中直接操作的数据库。相对于磁盘，内存的数据读写速度要高出几个数量级，将数据保存在内存中相 比从磁盘上访问能够极大地提高应用的性能。

**sys**库：通过这个数据库数据库，可以査询谁使用了最多的资源 基于IP或是用户。哪张表被访问过最多等信息。

### 注释
`-- `注意--后面还有个空格
`-- `后面的内容叫注释
`#`可以不跟空格，`#`后面的内容叫注释
有的时候`#`可能不能用，就得用`-- `

### 列出所有数据库
```mysql
SHOW DATABASES;
```

### 查看某一数据库里所有的表
#### 法一：
```mysql
USE databasename;
SHOW TABLES;
```
比如：
```mysql
USE mysql;
SHOW TABLES;
```

#### 法二：
> 把tables换成columns，from后改成表名，可以**查字段**
```mysql
SHOW TABLES FROM databasename;
```
比如：
```mysql
SHOW TABLES FROM mysql;
```

### select的特殊应用
#### 1. 查看时间
```mysql
SELECT NOW();
```
#### 2. 查看我们当前选择的库是哪个库
USE了哪个库查出来就是哪个库
```mysql
SELECT DATABASE();
```
#### 3. 查看版本
```mysql
SELECT VERSION();
```
#### 4. 查看当前登录数据库的用户
```mysql
SELECT USER();
```
#### 5. 查看数据路径
```mysql
SELECT @@datadir;#所有mysql的数据都存放在这个路径下
```
#### 6. 查看mysql安装路径
```mysql
SELECT @@basedir;
```
#### 7. 查看mysql安装的系统
```mysql
SELECT @@version_compile_os;
```

### select的正常应用
#### 1. 查询数据
`SELECT` 是 查询关键字
`*`代表查所有字段
```mysql
SELECT * FROM mysql.user; #查所有

SELECT HOST FROM mysql.users; #查host
```
#### 2. SHOW DATABASES的来源
```mysql
SELECT schema_name FROM information_schema.`SCHEMATA`;
```
	
#### 3. where条件
相当于`if`
```mysql
SELECT USER,HOST FROM mysql.user;
```
结果：
![[Pasted image 20250123175247.png]]
若我们要查询user为root的信息，则
```mysql
SELECT USER,HOST FROM mysql.user WHERE USER = 'root';
```
结果：
![[Pasted image 20250123175413.png]]

### 创建并查询、删除、修改等
#### 1. 创建库
如创建一个名为test的库：
```mysql
CREATE DATABASE test CHARSET utf8mb4;
```
#### 2. 创建表
如创建一个在test库中名为t1包含一个int类型字段id的表：
```mysql
USE test;
CREATE TABLE t1(id INT);
```
#### 3. 给表添加字段（修改）
如给t1表添加varchar类型字段name
```mysql
ALTER TABLE t1 ADD NAME VARCHAR(32);
```
#### 4. 查看表的属性（有哪几个字段，什么类型等）
如查看t1表的属性
```mysql
DESC t1;
```
结果：
![[Pasted image 20250123180634.png]]
#### 5. 删除表
如删除t1表：
```mysql
DROP TABLE t1;
```
#### 6. 插入数据
如给t1表插入id为1姓名为张三等的数据：
```mysql
INSERT INTO  t1 VALUES (1,"张三"),(2,"李四"),(3,"王五");
```
```mysql
SELECT * FROM t1; #已在test表中
```
结果：
![[Pasted image 20250123180933.png]]
#### 7. 查详细信息
```mysql
SELECT DATABASE();
SELECT * FROM information_schema.tables WHERE tableschema ='test';
SELECT * FROM information_schema.`COLUMNS` WHERE table_schema ='test';
```
#### 8. where and or
（下例只作演示）
```mysql
INSERT INTO t1 VALUES (4,"张三");#有出现同名情况
SELECT * FROM test.t1 WHERE NAME = '张三' AND id = 1;#指定id为1的张三
SELECT * FROM test.t1 WHERE NAME = '张三' OR 1 = 1;#sql注入中常用
```

#### 9. 删库
如删test库：
```mysql
DROP DATABASE test;
```

#### 10. 更改表名、字段名
- 修改表名：`ALTER TABLE 旧表名 RENAME TO 新表名;`
- 修改字段：`ALTER TABLE 表名 CHANGE 旧字段名 新字段名 新数据类型;`

### 联合查询
**将两个SELECT结果放在一个结果集里**
例如：
```mysql
SELECT * FROM test.t1 UNION SELECT 1,2;
```
结果：
![[Pasted image 20250123232908.png]]
**sql注入中可以用来查看暴露了几个字段，就是先从SELECT 1开始试，一个个往后加**

又如：
```mysql
SELECT USER,HOST FROM mysql.user UNION SELECT * FROM test.`t1`;
```
结果：
![[Pasted image 20250123233428.png]]

### handler语句——select被禁时可以用来查数据
#不用select如何查数据
可以实现select部分功能
 handler不是通用的SQL语句，是Mysql特有的，可以逐行浏览某个表中的数据，格式：
```
打开表：
HANDLER 表名 OPEN ;

查看数据：
HANDLER 表名 READ next;

关闭表：
HANDLER 表名 READ CLOSE;
```

### 预编译——绕过特定的字符过滤（如select）
#不用select如何查数据
预编译相当于定一个语句相同，参数不同的Mysql模板，我们可以通过预编译的方式，绕过特定的字符过滤
```
PREPARE 名称 FROM Sql语句 ? ;
SET @x=xx;
EXECUTE 名称 USING @x;
```
如：
```
1';Set @a=concat('sele','ct flag from `1919810931114514`;');prepare b from @a;execute b;# 拼接

1';Set @jie = 0x73656c656374202a2066726f6d20603139313938313039333131313435313460;PREPARE an from @jie;EXECUTE an;# 十六进制编码
```
步骤：
> set做好一盘菜——把你要执行的语句写到变量里；
> prepare把菜装到包装里——预编译这个变量里的SQL语句；
> execute出售你做好的预制菜——执行刚才预编译成功的SQL语句。

