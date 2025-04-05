---
date: 2025-01-17
tags:
  - web
  - WP
---
一打开就一张滑稽脸，开F12看看
![[Pasted image 20250117155155.png]]
发现有`source.php`文件，访问他看看
![[Pasted image 20250117155442.png]]
出现了php代码，可以进行审计，看到了`hint.php`我会想先访问一下
![[Pasted image 20250117155632.png]]
嗯……意思是flag在`ffffllllaaaagggg`这个文件里


<span style="color:rgb(255, 0, 0)">开始代码审计：</span>
#### 几个php函数：
**is_string()**：检测变量是否是字符串

**isset()**：检测变量是否已设置并且非 NULL

**in_array(要搜索的值，要搜索的数组)**：搜索数组中是否存在指定的值

**mb_substr($page，n，m)**：返回page中从第n位开始，到n+m位字符串的值

**mb_strpos()**：查找字符串在另一个字符串中首次出现的位置

**urldecode()**：将url编码后的字符串还原成未编码的样子

>下面是关于 `substr()、strpos()和in_array()`函数的详细介绍和用法：
（`mb_strpos和strpos`，`substr和mb_substr`在功能上几乎没什么区别）
#### strpos(string,find,start)函数：
**返回字符串在另一字符串中第一次出现的位置**，如果没有找到字符串则返回`FALSE`。
**注意**： 字符串位置是从 0 开始，不是从 1 开始。 
![[Pasted image 20250117164521.png]]

#### mb_substr(str,start,length,encoding)函数：
**返回字符串的一部分**，对于`substr()`函数，它只针对英文字符， 而`mb_substr()`对于中文也适用。
![[Pasted image 20250117165401.png]]

#### in_array(search,array,type)函数：
**搜索数组中是否存在指定的值**，找到值则返回`TRUE`，否则返回`FALSE`
![[Pasted image 20250117165546.png]]

大概的代码意思如下：
```php
 <?php//php固定起始格式
    highlight_file(__FILE__);//语法高亮，没什么特别的，不管
    class emmm//emmm是一个类
    {
        public static function checkFile(&$page)
        //checkFile是一个类中函数，讲得专业点，叫做“方法”。会接收一个变量page
        {
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];                 //白名单，类似于数组，前面是键，后面是值
            if (! isset($page) || !is_string($page)) {
            //如果变量为空或非字符串
                echo "you can't see it";
                return false;
            }
            if (in_array($page, $whitelist)) {
            //变量是否在白名单内
                return true;
            }
            $_page = mb_substr(     
                $page,      
                0,      
                mb_strpos($page . '?', '?')
                //‘.’是一个操作符，可以在page后面临时加一个‘?’     
            );//_page变量为page变量开头到第一个问号之间的部分
            if (in_array($_page, $whitelist)) {    
                return true;
            }//再次检测_page变量是否在白名单内
            $_page = urldecode($page);//对page进行url解码     
            $_page = mb_substr(    
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );//重复截取操作
            if (in_array($_page, $whitelist)) {
                return true;
            }//再次检测_page变量是否在白名单内
            //若到此处还未返回false或true，即函数未结束，则返回false
            echo "you can't see it";       
            return false;
        }
    }
    if (! empty($_REQUEST['file'])      
        && is_string($_REQUEST['file'])     
        && emmm::checkFile($_REQUEST['file'])
        //变量非空且是字符串且通过emmm checkFile      
    ) {
        include $_REQUEST['file'];
        //那么就包含（我个人的理解是“读取/加载”）并执行这个file表示的文件
        exit;       //结束了
    } else {        //如果上面的if条件不成立，就返回一张滑稽图
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }  
?>      //php固定末尾格式
```
1. 使用`in_array`函数对变量进行检测时，是检测与白名单中的**值**是否相同，与键无关
   
2. 要注意的是，当返回`ture或false`时，**函数将直接结束**；反之，当一直没有遇到`ture或false`，**函数会继续运行**
   
3. 在`mb_substr`函数中，**对page变量进行截取**，起始是0，即开头，长度为第一次出现?的位置（字符串位置是从0开始）。比如传入`hint.php`，临时加上问号后为`hint.php?`，从0开始算到?为8，返回8个字符即为`hint.php`。所以**若不主动添加?，将原封不动返回**；如果主动添加了?，则会**截取?中间的部分**，我们可以以此来构造payload，毕竟最后include的是整个page
   
4. payload为`file=hint.php?/../../../../ffffllllaaaagggg`，
   逻辑：将file传入后，是字符串且非空，不满足第一个`if`，没有`return`；
   `file`的值不在白名单内，不满足第二个`if`，没有`return`；
   而后进行截取操作，被截为成`hint.php`，满足第三个`if`，`return true`，退出函数，进行`include`；
   后面的`../`可以一个个加试出来，
   也有一种解释是：我们当前的`source.php`一般是在`html`目录下，往上是`www`，`var`，然后到`根目录`，**flag一般就放在根目录下面**，这里还有一个`hint.php?`，因此需要返回四层才能到根目录。

5. 这就发现了**include的函数的一个关键特性**，会在传入字符串的末尾**找有效的路径来执行**（本来读取hint.php，然后出了个问号把这个路径破坏掉了。于是继续重新读取路径，就读到了后面的`/../../../../ffffllllaaaagggg`） ^49a85b

![[Pasted image 20250117172755.png]]
flag{320c4048-795f-48a4-89b3-268224ab08ee}