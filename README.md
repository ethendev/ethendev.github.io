# About this Jekyll Blog Theme


If you like this blog theme, please give me a star.

## Content

* [Preview](#preview)
* [Usage](#usage)
    * [1. Install ruby and jekyll environment](#1-install-ruby-and-jekyll-environment)
    * [2. Clone my code](#2-clone-my-code)
    * [3. Change parameter](#3-change-parameter)
    * [4. Write post](#4-write-post)
    * [5. Local launch](#5-local-launch)
* [License](#license)

## Preview

First, take a look at the blog page.

![index](/img/post.png)


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


Change your title and url.

```yml
# Site settings
title: BlogHub
brief-intro: Back-end Dev Engineer
baseurl: "" # the subpath of your site, e.g. /blog
url: "https://ethendev.github.io/"
```


If you want to comment, visit https://disqus.com/ or https://livere.com/ to get your own name, and replace the below value.

```yml
# comments
livere_uid: XXXX
disqus_shortname: xxxx
```


Statistic and Analysis

This theme supports Google Analytics and Baidu Statistics，visit https://www.google.com/analytics/ or http://tongji.baidu.com/, get your own id.

```yml
# statistic analysis
# peplace your own ID
baidu_tongji_id: xxxxxxxxxxxx
google_analytics_id: UA-xxxxxxxx
```

### 4. Write post

To create a new post, all you need to do is create a file in the _posts directory. How you name files in this folder is important. Jekyll requires blog post files to be named according to the following format:
```
yyyy-MM-dd-title.md
```

Before you start posting, you should declare layout、title、categories、tags、author(optional) info.


```
---
layout: post
title: SpringBoot中数据源读写分离配置
tags:  [SpringBoot, 读写分离, DB]
categories: [SpringBoot]

---
```

These follow code is for generating table of contents.
```
* content
{:toc}
```


Each post automatically takes the first block of text, from the beginning of the content to the first occurrence of excerpt_separator, and sets it in the post’s excerpt. 

You can set `excerpt_separator` in the `_config.yml`, as follows:

```yml
# excerpt
excerpt_separator: "\n\n\n\n"
```

If you don’t like the automatically-generated post excerpt, it can be explicitly overridden by adding an excerpt value to your post’s YAML Front Matter. 

```
---
layout: post
title: 
tags:  
categories: 
author: Ethen
excerpt: your excerpt
---
```

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

Visit http://127.0.0.1:4000/ to see your blog.


## License

[MIT License](https://github.com/ethendev/ethendev.github.io/blob/master/LICENSE.md)
