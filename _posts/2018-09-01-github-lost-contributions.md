---
layout: post
title: 修改GitHub邮箱后丢失contributions的解决办法
tags:  [GitHub]
categories: [GitHub]
keywords: GitHub,Email,contributions
---



今天修改了 GitHub 邮箱并且将旧邮箱删除。后来查看 GitHub 主页的时候，发现自己的 contributions 数据丢失了。原来 contributions 的统计是和 Email 关联的，修改了邮箱后数据就没有了。




![contributions](https://i.loli.net/2018/09/01/5b8aa74678edf.png)

### 解决办法

查看当前的邮箱地址
```
$ git config user.email
```
 
修改为新的邮箱
```
$ git config --global user.email "xxx@example.com"
```

修改了邮箱后发现原来的 contributions 还是没有，但是修改邮箱之后提交的记录有了。


在 GitHub 官网 [github help](https://help.github.com/articles/changing-author-info/#platform-windows) 找到了解决办法。


1、创建新的 clone 仓库, 将 repo.git 修改为你的仓库名字
```
git clone https://github.com/user/repo.git
cd repo.git
```

2、修改 Git 历史

GitHub 提供了一个脚本来修改历史记录，不过由于 gist 被墙，没有梯子的话看不到这个脚本。 脚本内容如下：

```
#!/bin/sh

git filter-branch --env-filter '

OLD_EMAIL="your-old-email@example.com"
CORRECT_NAME="Your Correct Name"
CORRECT_EMAIL="your-correct-email@example.com"

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

将上面的 OLD_EMAIL、CORRECT_NAME 、CORRECT_EMAIL 修改为自己对应的邮箱和用户名，然后运行脚本。


3、使用`git log`查看新 Git 历史有没有错误

4、把正确的历史提交到 GitHub
```
git push --force --tags origin 'refs/heads/*'
```

执行完上面的操作后，原先丢失的数据又回来了。
![contributions](https://i.loli.net/2018/09/01/5b8aa7469165a.png)