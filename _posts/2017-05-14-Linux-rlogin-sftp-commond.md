---
layout: post
title: Linux sftp、rlogin命令中执行一段脚本的方法
tags:  [Linux, sftp, rsh]
categories: [Linux]
keywords: Linux,sftp,rsh
---


Linux中使用rsh远程登陆或者sftp的时候，想要在登陆命令后面执行一段命令，可以使用下面的方式。




### sftp
使用案例：
```bash
sftp user@servedr <<EOF
    ls
    put test.txt
EOF
```

### rsh
```bash
rsh -l irteam $HOST "cd apps/apache-tomcat-7.0.78/bin; ./deploy.sh"
```