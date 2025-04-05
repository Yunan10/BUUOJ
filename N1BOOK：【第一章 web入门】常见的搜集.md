用dirsearch-master扫目录
`python dirsearch.py -u 网址 -e*`

扫出来几个绿的康康：
/.DS_Store   下载后为找到与flag相关内容
/.index.php.swp   下载后有与flag3相关内容`flag3:p0rtant_hack}`
/robots.txt  访问得到/flag1_is_her3_fun.txt，再访问得到`flag1:n1book{info_1`
/index.php~  访问得到`flag2:s_v3ry_im`
拼接起来就是flag：`n1book{info_1s_v3ry_imp0rtant_hack}`

但是……
发现用dirsearch扫不出`/index.php~`
好吧那只能通过敏感文件备份文件的经验自己去尝试
首先robots.txt肯定不成问题
剩下两个就当积累了
加`~`是指临时文件
`.swp`是意外退出的备份文件

