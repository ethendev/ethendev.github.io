# About this Jekyll Blog Theme

[![GitHub stars](https://img.shields.io/github/stars/ethendev/ethendev.github.io.svg)](https://github.com/ethendev/ethendev.github.io/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/ethendev/ethendev.github.io.svg)](https://github.com/ethendev/ethendev.github.io/network)
[![GitHub issues](https://img.shields.io/github/issues/ethendev/ethendev.github.io.svg)](https://github.com/ethendev/ethendev.github.io/issues)
[![GitHub release](https://img.shields.io/github/release/ethendev/ethendev.github.io.svg)](https://github.com/ethendev/ethendev.github.io/releases)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/ethendev/ethendev.github.io/master/LICENSE)


If you like this blog theme, please give me a star.

## Content

* [Preview](#preview)
* [Page Details](#page-details)
    * [Home](#home)
    * [Archives](#archives)
    * [Categories](#categories)
    * [Tags](#tags)
    * [About](#about)
    * [Comments](#comments)
    * [Post Contents](#post-contents)
    * [Code Highlight](#code-highlight)
    * [Light Shadow](#light-shadow)
    * [Mobile Adaptation](#mobile-adaptation)
    * [Footer](#footer)
    * [Statistical Analysis](#statistical-analysis)
* [Usage](#usage)
    * [1. Install ruby and jekyll environment](#1-install-ruby-and-jekyll-environment)
    * [2. Copy theme code](#2-copy-theme-code)
    * [3. Change parameter](#3-change-parameter)
        * [Basic info](#basic-info)
        * [Link info](#link-info)
        * [Comments info](#comments-info)
        * [Statistical analysis info](#statistical-analysis-info)
    * [4. Write post](#4-write-post)
    * [5. Local launch](#5-local-launch)
    * [6. Push to GitHub](#6-push-to-github)
* [License](#license)

## Preview

First, take a look at the blog page.

![index](http://ww3.sinaimg.cn/large/7011d6cfjw1f3bdli86awj211k0oyqen.jpg)


## Usage


### 1. Install ruby and jekyll environment

This step and Step 5 mainly talk to you how to launch blog at local. If you don't want to launch at local, you can ignore these 2 steps. But I still strongly suggest to do this. Ensure there is nothing wrong before pushing to the github.

The Windows users can directly use [RubyInstaller](http://rubyinstaller.org/) to install ruby environment. Follow the prompts while installing.

Install jekyll commands:

```
gem install jekyll
```

For more details, you can view the jekyll official website. [https://jekyllrb.com/](https://jekyllrb.com/)

### 2. Clone my code

```
git clone https://github.com/ethendev/ethendev.github.io.git
```

### 3. Change parameter

Mainly change the parameters at file `_config.yml` and use your own `favicon.ico`.

#### Basic info

Shows at site header part.

```yml
# Site settings
title: BlogHub
brief-intro: Back-end Dev Engineer
baseurl: "" # the subpath of your site, e.g. /blog
url: "https://ethendev.github.io/"
```

#### Comments info

Get your own `short_name`:

Visit https://disqus.com/ or http://duoshuo.com/. And follow the prompts at the site.

```yml
# comments
# two ways to comment, only choose one, and use your own short name
duoshuo_shortname: #hygblog
disqus_shortname: xxxx
```

When you done, you can also see the comments info at disqus or duoshuo admin console.


### Statistic and Analysis

This theme supports Google Analytics and Baidu Statistics，visit https://www.google.com/analytics/ or http://tongji.baidu.com/, get your own id.

```yml
# statistic analysis
# peplace your own ID
baidu_tongji_id: xxxxxxxxxxxx
google_analytics_id: UA-xxxxxxxx
```

### 4. Write post

You can write posts at folder `_posts`. Before you start posting, you should declare layout、title、date、categories、tags、author(optional) info、mathjax(optional，click [here](https://www.mathjax.org/) for more detail about `Mathjax`).

```
---
layout: post
title:  "yout title"
date:   2016-03-12 11:40:18 +0800
categories: jekyll
tags: jekyll
author: ethendev
mathjax: true
---
```

These follow code is for making contents.
```
* content
{:toc}
```

You can use 4 wraps as a excerpt separator. The words before separator as excerpt show in the index page. When you enter the post page, you can read full article.

The wraps config is in the file `_config.yml`, as follows:

```yml
# excerpt
excerpt_separator: "\n\n\n\n"
```

You should use markdown syntax to write article, just like write readme in github.

You can use 3 \`\`\` to write code block.

### 5. Local launch

use command:

```
jekyll server
```

When you see the content below, it starts to work.

```
Server address: http://127.0.0.1:4000/
Server running... press ctrl-c to stop.
```

Visit http://127.0.0.1:4000/ to see your blog!!!

### 6. Push to GitHub

If there is nothing wrong, push your code to github!


## License

[MIT License](https://github.com/ethendev/ethendev.github.io/blob/master/LICENSE.md)
