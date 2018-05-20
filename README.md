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

These follow code is for generating table of contents.
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


## License

[MIT License](https://github.com/ethendev/ethendev.github.io/blob/master/LICENSE.md)
