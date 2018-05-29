---
layout: post
title: Jekyll笔记
categories: [Jekyll]
tags: [Jekyll]
keywords: Jekyll,语法
---

* content
{:toc}


### 目录结构
Jekyll 的核心其实是一个文本转换引擎。它的概念其实就是： 你用你最喜欢的标记语言来写文章，可以是 Markdown，也可以是 Textile,或者就是简单的 HTML, 然后 Jekyll 就会帮你套入一个或一系列的布局中。在整个过程中你可以设置URL路径, 你的文本在布局中的显示样式等等。这些都可以通过纯文本编辑来实现，最终生成的静态页面就是你的成品了。




一个基本的 Jekyll 网站的目录结构一般是像这样的：
```
.
├── _config.yml
├── _drafts
|   ├── begin-with-the-crazy-ideas.textile
|   └── on-simplicity-in-technology.markdown
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts
|   ├── 2007-10-29-why-every-programmer-should-play-nethack.textile
|   └── 2009-04-26-barcamp-boston-4-roundup.textile
├── _data
|   └── members.yml
├── _site
└── index.html
```

其作用如下：

| 文件/目录        | 描述   |
| --------   | :-----  |
| _config.yml     | 保存配置数据。很多配置选项都会直接从命令行中进行设置，但是如果你把那些配置写在这儿，你就不用非要去记住那些命令了。 |
| _drafts        |   drafts 是未发布的文章。这些文件的格式中都没有 title.MARKUP 数据。|
| _includes        |   你可以加载这些包含部分到你的布局或者文章中以方便重用。可以用这个标签  {% include file.ext %} 来把文件 _includes/file.ext 包含进来。  |
| _layouts | layouts 是包裹在文章外部的模板。布局可以在 YAML 头信息中根据不同文章进行选择。 这将在下一个部分进行介绍。标签  {{ content }} 可以将content插入页面中。 |
| _posts | 这里放的就是你的文章了。文件格式很重要，必须要符合: YEAR-MONTH-DAY-title.MARKUP。 The permalinks 可以在文章中自己定制，但是数据和标记语言都是根据文件名来确定的。 |
| _data | 格式良好的网站数据应该放在这里。 jekyll引擎会自动加载该目录中的所有yaml文件（以.yml或.yaml结尾）。 如果目录下有文件members.yml，则可以通过site.data.members访问该文件的内容。|
| _site | 一旦 Jekyll 完成转换，就会将生成的页面放在这里（默认）。最好将这个目录放进你的 .gitignore 文件中。 |
| index.html and other HTML, Markdown, Textile files | 如果这些文件中包含 YAML 头信息 部分，Jekyll 就会自动将它们进行转换。当然，其他的如 .html， .markdown，  .md，或者 .textile 等在你的站点根目录下或者不是以上提到的目录中的文件也会被转换。 |
| Other Files/Folders | 其他一些未被提及的目录和文件如  css 还有 images 文件夹， favicon.ico 等文件都将被完全拷贝到生成的 site 中。 这里有一些使用 Jekyll 的站点，如果你感兴趣就来看看吧。 |


### 基本用法
安装了 Jekyll 的 Gem 包之后，就可以在命令行中使用 Jekyll 命令了。有以下这些用法：
```
jekyll build
# => 当前文件夹中的内容将会生成到 ./site 文件夹中。

jekyll build --destination <destination>
# => 当前文件夹中的内容将会生成到目标文件夹<destination>中。

jekyll build --source <source> --destination <destination>
# => 指定源文件夹<source>中的内容将会生成到目标文件夹<destination>中。

jekyll build --watch
# => 当前文件夹中的内容将会生成到 ./site 文件夹中，
#    查看改变，并且自动再生成。
```

Jekyll 同时也集成了一个开发用的服务器，可以让你使用浏览器在本地进行预览。
```
jekyll serve
# => 一个开发服务器将会运行在 http://localhost:4000/

jekyll serve --detach
# => 功能和`jekyll serve`命令相同，但是会脱离终端在后台运行。
#    如果你想关闭服务器，可以使用`kill -9 1234`命令，"1234" 是进程号（PID）。
#    如果你找不到进程号，那么就用`ps aux | grep jekyll`命令来查看，然后关闭服务器。[更多](http://unixhelp.ed.ac.uk/shell/jobz5.html).

jekyll serve --watch
# => 和`jekyll serve`相同，但是会查看变更并且自动再生成。
```

### 模板语法
#### 使用变量

所有的变量是都一个树节点, 比如模板中定义的头部变量, 需要使用下面的语法获得

page.title

page 是当前页面的根节点。其中全局根结点有

* site _config.yml 中配置的信息
* page 页面的配置信息
* content 模板中,用于引入子节点的内容
* paginator 分页信息

#### site 下的变量
site.time 运行 jekyll 的时间
site.pages 所有页面
site.posts 所有文章
site.related_posts 类似的10篇文章,默认最新的10篇文章,指定lsi为相似的文章
site.static_files 没有被 jekyll 处理的文章,有属性 path, modified_time 和 extname.
site.html_pages 所有的 html 页面
site.collections 新功能,没使用过
site.data _data 目录下的数据
site.documents 所有 collections 里面的文档
site.categories 所有的 categorie
site.tags 所有的 tag
site.[CONFIGURATION_DATA] 自定义变量

#### page 下的变量
page.content 页面的内容
page.title 标题
page.excerpt 摘要
page.url 链接
page.date 时间
page.id 唯一标示
page.categories 分类
page.tags 标签
page.path 源代码位置
page.next 下一篇文章
page.previous 上一篇文章

#### paginator 下的变量
paginator.per_page 每一页的数量
paginator.posts 这一页的数量
paginator.total_posts 所有文章的数量
paginator.total_pages 总的页数
paginator.page 当前页数
paginator.previous_page 上一页的页数
paginator.previous_page_path 上一页的路径
paginator.next_page 下一页的页数
paginator.next_page_path 下一页的路径

#### Liquid
jekyll使用Liquid语言，Liquid 是一门开源的模板语言。[Liquid 模板语言中文文档](https://liquid.bootcss.com/)
