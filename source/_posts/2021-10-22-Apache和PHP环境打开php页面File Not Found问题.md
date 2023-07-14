---
layout:     post
title:      Apache和PHP环境打开php页面File Not Found问题
subtitle:   一些无聊的小问题。
date:       2021-10-22
author:     YSY
header-img: img/404-bg.jpg
catalog: true
tags:
    - 问题不大
    - 后端
---

### 问题

之前搞了个腾讯云的轻量应用服务器（预装环境LAMP）来玩，结果最近发现网站目录下面的php文件访问不了，在浏览器打开就出现“File Not Found”的提示。

搜罗了很多答案，没有一个明确能解决问题的，不过还是得到了一些启示。下面属于我的个例，不一定能解决所有此类问题。

### 解决

腾讯云的这种服务器预装的软件都在此目录下面，包括相关配置：

```bash
[root@VM-0-15-centos ~]# cd /usr/local/lighthouse/softwares
[root@VM-0-15-centos softwares]# ls
apache  mariadb  oniguruma  php
```

1、我们先配置一下apache，让你的网站首页可以以index.php的形式存在：

```bash
# 直接编辑配置文件
vim apache/conf/httpd.conf
```

然后找到这段内容：

```xml
<IfModule dir_module>
    DirectoryIndex index.html index.php
</IfModule>
```

原本只有index.html，在后面补充即可，空格分隔。

2、然后解决php文件“File Not Found”的问题，很多解答说有配置的关系，或者网站目录路径没设置对，或者SELinux安全问题等等，但最终都没得到解决。因为我这个环境是LAMP预装，各种配置肯定没有什么大差错。查看php-fpm进程也正常运行：

```bash
[root@VM-0-15-centos ~]# ps -ef | grep php
root      721409       1  0 15:31 ?        00:00:00 php-fpm: master process (/usr/local/lighthouse/softwares/php/etc/php-fpm.conf)
daemon    721410  721409  0 15:31 ?        00:00:00 php-fpm: pool www
daemon    721411  721409  0 15:31 ?        00:00:00 php-fpm: pool www
root      730570  727734  0 16:36 pts/1    00:00:00 grep --color=auto php
```

后来经过尝试，发现还是Apache服务配置的问题，同样地，还是修改刚才的 `httpd.conf` 文件，找到 **ProxyPassMatch** 这一行内容，我的在文件末尾：

```bash
ProxyPassMatch ^/(.*\.php(/.*)?)$ unix:/usr/local/lighthouse/softwares/apache/logs/php-fpm.sock|fcgi://127.0.0.1/home/www/htdocs/
```

直接注释掉或者删掉，然后保存文件。

最后重启一下Apache服务：

```bash
service apache restart
```

就这样解决了。

