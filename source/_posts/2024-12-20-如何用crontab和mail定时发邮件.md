---
title: 如何用crontab和mail定时发邮件
date: 2024-12-20 15:51:19
tags:
    - 鼓捣折腾
    - Linux
    - 后端
---

### 背景

虽然说懒人炒股不一定能赚到钱，但是懒人自有偷懒的办法。

最近突发奇想，想在服务器上设置一个定时任务，每隔一段时间查询某只股票的股价，如果达到告警条件，就自动发一封邮件给指定邮箱。正好QQ邮箱还接入了Android的系统级推送，这样我就不用每天盯盘了。

### 准备

先准备一个163邮箱，其他邮箱也行（支持开启SMTP服务即可），这里以163邮箱为例。

登录网页版，找到“设置”-“POP3/SMTP/IMAP”，然后点击“新增授权码”，复制这个授权码先暂存，等会儿要用。

### 基础设施

我的小水管服务器是Ubuntu 16.04，所以下面就基于这个环境来搞。

为了能用命令发邮件，我们先安装相关的依赖：

```shell
apt install mailutils
```

首次安装后，会出现一个粉紫色界面，提示你配置“Postfix Configuration”，这里直接选择“No configuration”，然后OK。

接着安装ssmtp服务：

```shell
apt install ssmtp
```

安装完后编辑此配置文件 `vim /etc/ssmtp/ssmtp.conf`，添加并保存如下内容：

```
root=123456@163.com
mailhub=smtp.163.com:465
AuthUser=123456@163.com
AuthPass=xxxxxx
UseTLS=Yes
```

注意这里的123456@163.com是163邮箱的账号，xxxxxx是刚才复制的授权码。

再编辑 `/etc/ssmtp/revaliases`，添加并保存如下内容：

```
root:123456@163.com:smtp.163.com:465
```

这里的root是Ubuntu当前登录的用户名，如果你不是root用户，可能是别的名字。

OK，配置完之后我们发一个测试邮件，看看能不能成功：

```shell
echo "测试内容" | mail -s "测试标题" 任意邮箱地址@qq.com
```

如果成功，你就可以在对应的目标邮箱中收到刚才配置的163邮箱发过来的邮件。

记得再安装一些脚本小工具，方便等会儿解析json和计算股价用：

```shell
apt install jq bc
```

### 脚本实现

接下来，我们就可以写脚本来查询股价和定时发送邮件了。

比如我要实现一个脚本，通过免费的API查询数据，每5分钟检查一次，当特斯拉股价低于400或者高于500时，发一封邮件给我的QQ邮箱。

> 我们先去 [www.alphavantage.co](https://www.alphavantage.co) 注册一个账号，然后申请一个API key。 

直接上Shell脚本（其中的yyyyyy要换成你自己申请的API key）：

```shell
#!/bin/bash

API_KEY="yyyyyy"
STOCK_SYMBOL="TSLA"
LOW_THRESHOLD=400
HIGH_THRESHOLD=500
RECIPIENT_EMAIL="任意邮箱地址@qq.com"

get_stock_price() {
    response=$(curl -s "https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=$STOCK_SYMBOL&interval=5min&apikey=$API_KEY")
    price=$(echo $response | jq -r '.["Time Series (5min)"] | to_entries | .[0].value["4. close"]')
    
    # 检查价格是否为空或不是数字
    if [[ -z "$price" || ! "$price" =~ ^[0-9]+(\.[0-9]+)?$ ]]; then
        echo "Error: Unable to fetch stock price"
        return 1  # 返回错误状态
    fi
    
    echo $price
    return 0  # 返回成功状态
}

send_email() {
    echo "Tesla stock price alert: Current price is $1" | mail -s "Tesla Stock Alert" $RECIPIENT_EMAIL
}

# 获取股价并检查是否成功
stock_price=$(get_stock_price)
if [ $? -eq 0 ]; then
    if (( $(echo "$stock_price < $LOW_THRESHOLD" | bc -l) )) || (( $(echo "$stock_price > $HIGH_THRESHOLD" | bc -l) )); then
        send_email $stock_price
    else
        echo "Stock price $stock_price is within the normal range ($LOW_THRESHOLD - $HIGH_THRESHOLD). No email sent."
    fi
else
    echo "No email sent due to error in fetching stock price."
fi
```

保存脚本，随便命名为 `check_tesla_stock.sh`，然后给予可执行权限：

```shell
chmod +x check_tesla_stock.sh
```

可以先手动执行脚本测试一下，确保它能正常发送邮件（当然，不要忘了当前特斯拉股价可能并不在你的告警范围，所以测试时修改一下阈值吧）。

最后一步，用crontab添加定时任务即可：

```shell
crontab -e
```

进入编辑模式后，在文件最开头加入：

```
MAILTO=""
```

目的是为了防止crontab把脚本中不需要的echo输出发送到你自己的163邮箱里面（这是crontab本身的特性，会联动mail命令，并非是脚本中的echo命令会发邮件，请不要误解），因为我们实际上只需要脚本中 `send_email $stock_price` 这行命令的执行，所以你把其他无关的echo删掉也没关系。

再在末尾添加如下内容后保存：

```
*/5 * * * * /你脚本所在的具体路径/check_tesla_stock.sh
```

大功告成！如此这般，定时任务就设置好了。静待特斯拉股价上天入地，邮件就会自动发过来了。
