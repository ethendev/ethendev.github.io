---
layout: post
title: 自动切割 Nginx 日志
tags:  [Nginx]
categories: [Nginx]
keywords: Nginx,log,rotat
---


默认nginx不会自动切割日志，当日志文件越来越大当时候，不仅浪费磁盘空间，nginx的性能也会降低。可以使用Linux的logrotate来解决这个问题。




### logrotate
logrotate 可以自动对日志进行切割、压缩和删除。而且自动化处理，不需要人为操作，使用非常方便。

#### 配置 logrotate
进入 logrotate.d 目录
```
cd /etc/logrotate.d/
```

创建 nginx 文件，并写入如下内容
```
/test/nginx-1.14.2/logs/*.log
{
    daily
    rotate 30
    missingok
    dateext
    compress
    delaycompress
    notifempty
    sharedscripts
    postrotate
        if [ -f /test/nginx-1.14.2/logs/nginx.pid ]; then
            kill -USR1 `cat /test/nginx-1.14.2/logs/nginx.pid`
        fi
    endscript
}
```

调试脚本，测试脚本是否正确
```
sudo /usr/sbin/logrotate -d -f /etc/logrotate.d/nginx
```

手动运行运行脚本分割日志
```
sudo /usr/sbin/logrotate -f /etc/logrotate.d/nginx
``` 

#### logrotate 参数说明

|配置|说明|
|--|--------|
|daily|指定转储周期为每天|
|weekly|指定转储周期为每周|
|monthly|指定转储周期为每月|
|rotate count|指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份|
|compress|通过gzip 压缩转储以后的日志|
|nocompress|不做gzip压缩处理|
|create mode owner group|轮转时指定创建新文件的属性，如create 0777 nobody nobody|
|nocreate|不建立新的日志文件|
|delaycompress|和compress 一起使用时，转储的日志文件到下一次转储时才压缩|
|nodelaycompress|覆盖 delaycompress 选项，转储同时压缩|
|missingok|如果日志丢失，不报错继续滚动下一个日志|
|ifempty|即使日志文件为空文件也做轮转，这个是logrotate的缺省选项|
|notifempty|当日志文件为空时，不进行轮转|
|mail address|把转储的日志文件发送到指定的E-mail 地址|
|olddir directory|转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统|
|noolddir|转储后的日志文件和当前日志文件放在同一个目录下|
|sharedscripts|运行postrotate脚本，作用是在所有日志都轮转后统一执行一次脚本。如果没有配置这个，那么每个日志轮转后都会执行一次脚本|
|prerotate|在logrotate转储之前需要执行的指令，例如修改文件的属性等动作；必须独立成行|
|postrotate|在logrotate转储之后需要执行的指令，例如重新启动 (kill -HUP) 某个服务！必须独立成行|
|dateext|使用当期日期作为命名格式|
|dateformat .%s|配合dateext使用，紧跟在下一行出现，定义文件切割后的文件名，必须配合dateext使用，只支持 %Y %m %d %s 这四个参数|
|size(minsize) log-size|当日志文件到达指定的大小时才转储，log-size能指定bytes(缺省)及KB (sizek)或MB(sizem)，例如 size 100M|

### 添加定时任务

通过下面的命令添加定时任务(注意需要指定运行 nginx 的用户，不然可能没有权限无法正确执行)
```
sudo crontab -u root -e
```

```
# rotate nginx log erery day
0 0 * * * /usr/sbin/logrotate -f /etc/logrotate.d/nginx
```

如果要删除上面的定时任务，运行如下命令

```
crontab -r
```

如果任务没有正确执行，可以通过如下命令查看任务日志
```
vi /var/log/cron
```
