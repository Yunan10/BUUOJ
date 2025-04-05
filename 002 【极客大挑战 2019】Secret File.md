![[Pasted image 20250210115223.png]]
蒋璐源是谁我也不认识……先开源代码看看有没有有用的
注意到：
![[Pasted image 20250210115812.png]]
访问一下：
![[Pasted image 20250210115957.png]]
点了之后：
![[Pasted image 20250210120014.png]]
F12查看：
![[Pasted image 20250210120043.png]]
一闪而过的应该是`action.php`
BP抓包看看
![[Pasted image 20250210120539.png]]
注释掉了这个文件`secr3t.php`
访问看看
![[Pasted image 20250210120636.png]]
> `../`：防止目录遍历攻击。
> `tp`（不区分大小写）：可能意图过滤协议如`phar://`或`http://`。
> `input`/`data`（不区分大小写）：阻止`php://input`或`data://`协议，防止执行任意代码。
 
`http://7e65c09f-cc1f-48ff-809d-d800a3449f13.node5.buuoj.cn:81/secr3t.php?file=php://filter/read=convert.base64-encode/resource=flag.php`
把回显的数据base64解码一下
![[Pasted image 20250210121634.png]]
