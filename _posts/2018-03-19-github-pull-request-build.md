---
layout: post
title: GitHub pull request时Jenkins自动构建教程
tags:  [GitHub, Jenkins]
categories: [Jenkins]
keywords: GitHub,pull request,Jenkins
---

* content
{:toc}

当开发人员向GitHub的master分支提交pull request时，需要相关的人员进行review后，才merge到master分支。通过Jenkins，可以很方便的实现pull request时自动触发构建、测试代码，极大的提高工作效率。下面简单介绍一下配置步骤。




## 1.生成Token
GitHub上点击个人头像，依次选择Settings > Developer settings > Personal access tokens, 点击Generate new token按钮进入新Token生成页面。
![Generate new token](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/FFF633F1444648EB86741F2BCB382B4F.png)


勾选下图所示的选项，然后单击页面下方的Generate token按钮生成Token.
![Select scopes](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/81D30E0A2C774D28952E7716C7728C30.png)

注意：将生成的Token复制保存，后面你就无法再看到它了。
![Personal access tokens](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/64CDB652C25C4E0984FE1A949DBD2051.png)


## 2.安装插件
Jenkins > 系统管理 > 管理插件 > 可选管理，搜索GitHub Pull Request Builder，然后选择直接安装
![Install Plugin](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/2214F18219DD491283759868FF44A7F9.png)

插件介绍地址 [https://github.com/jenkinsci/ghprb-plugin/raw/master/README.md](https://github.com/jenkinsci/ghprb-plugin/raw/master/README.md)  。


## 3. 配置GitHub Pull Request Builder
在Jenkins中添加Credentials，Kind处选择Secret text, Secret处填写GitHub的Token，后面配置GitHub Pull Request Builder时需要用到它。
![Add Credentials](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/BE6C21CA564A4887BA339A064CA1AA4A.png)

然后再使用GitHub的用户名和密码创建另一个Credentials，后面配置任务的时候会用到。当然如果你已经添加过了，就不用再配置。
![Add Credentials](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/C4A3EBB175544E8F8DF015CE91CBE9A7.png)

然后在 jenkins > 系统管理 > 系统设置 > GitHub Pull Request Builder如下配置
![Config GitHub Pull Request Builder](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/83C218FB31D04C688807A1221C508543.png)

GitHub Server API URL: 默认为 https://api.github.com , 企业版填写 https://域名/api/v3/。  
Credentials: 选择之前用GitHub生成的token创建的Credentials。


填好上面的内容后，可以通过下方的Connect to API 按钮验证连接GitHub是否正常，Check repo permissions按钮验证权限。

## 4. 配置任务
### 4.1 新建任务
新建一个自由风格的任务，然后配置该任务
![Create Job](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/75FC8FCF52CF49DBAFB288678B9506CE.png)

将项目的GitHub URL添加到GitHub project处(可以输入到浏览器中的项目，例如https://github.com/janinko/ghprb)
![Config GitHub URL](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/015E2F6A6FED484B8D1D4D53E535E5F3.png)

### 4.2 源码管理
在配置源码管理中添加代码在GitHub中的地址，Credentials处选择上面用GItHub的账号和密码创建的Credentials。
![SC Management](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/28C48CB27D1749A58B037E5B01407501.png)

* refspec：
  * 如果只想构建PR，请将refspec设置为`+refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}`
  * 如果你想构建PR和分支，请将refspec设置为 `+refs/heads/*:refs/remotes/origin/* +refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/`
* Branch Specifier: 填写 `${sha1}`, 如果想要用到提交的pr，则这个地方填写 `${ghprbActualCommit}`

### 4.3 构建触发器
在构建触发器中添加管理员名单，勾选Use github hooks for build triggering并点击后面的高级按钮，White list中添加针对这个任务允许进行pull request的GitHub用户账号。
![Build Trigger](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/07932EB674094D5A8649895FA99A19E5.png)

### 4.4 构建步骤
构建这里根据自己的项目情况选择合适的构建工具，我的项目使用Gradle构建的所以选择的是Invoke Gradle script。
![Build](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/E6698CF4937448D68096E86034DCE72C.png)

如果添加了Set build status to "pending" on GitHub commit这个构建步骤，那么在构建的时候GItHub项目的Pull requests中会显示pending状态。  
![Build Status](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/68712891AFB640428AE0403CF3A4C7F0.png)

## 5. 新建webhook
在GitHub对应的项目远程仓库，Settings > Webhooks中，新建一个webhook, 按下图填写Jenkins地址
![Add webhooks](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/353B2D0D887344708510088ADE3E4FC6.png)

选中Let me select individual events，勾选Pull requests选项。
![individual events](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/1E2E92499FE44F55991ED5F8480DA0A3.png)

如果GitHub能够正确连接Jenkins，那么Recent Deliveries下面会有一条没有报错的记录。
![Recent Deliveries](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/D6CDCBF83E734A369773A9170EEE1819.png)

到此，配置就完成了。下面往项目的master分支发一个pull request测试。发PR后GitHub上显示pending check状态
![pending check](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/68712891AFB640428AE0403CF3A4C7F0.png)

Jenkins上成功触发了一次构建  
![Jenkins Building](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/B528D27684D94F568C2825746700CA37.png)

当成功构建后，GitHub上面的状态修改成successful check。
![successful check](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/D9D968CF99AD4236A4116FA48D5C4EF2.png)

## 6. 常见问题
### 6.1 构建失败
![GitHub Build Failed](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/CCA7A2A3532F4C969E745C3C3157A4D5.png)  
可以通过上图中构建结果右侧的Details打开Jenkins上对应的任务查看Build History，通过`控制台输出`可以看到报错信息，根据信息提示修改配置。

![Jenkins Build Failed](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/DB21B1200FE2442390EC3460B3BA82FE.png)

### 6.2 PR时无法触发Jenkins构建
一般是配置有问题，可以通过看日志信息排查  
在GitHub Webhooks的 Manage webhook 页面最下方的Recent Deliveries查看GitHub的错误信息。
![Recent Deliveries](https://github.com/ethendev/data/raw/master/silo/img/github_jenkins/7A27246094FE4FA2A8068CF6ADBF0976.png)

Jenkins > 系统管理 > 日志管理， 打开 所有系统日志 查看错误信息  

