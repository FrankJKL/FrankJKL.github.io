---
layout: post
title:  "在Jekyll博客添加gitment评论系统"
categories: 博客构建
tags: Gitment
author: tobyjiang
typora-root-url: ..
comments: true
date: 2019-01-08 00:00:01 +0000
---

* content
{:toc}
本文章转载于http://xichen.pub/2018/01/31/2018-01-31-gitment/，并在其基础上进行完善

## 1、在[setting - OAuth Application 注册页面](https://github.com/settings/applications/new)完成注册

```
Application Name: gitment评论 //随便填
Homepage Url: https://frankjkl.github.io //博客的域名
Application description: //随便填，留空也可以
Authorization Callback URL: https://frankjkl.github.io //一定要写自己Github Pages的URL
```

注册成功后会得到`Client ID`和`Client Secret`




## 2、在jekyll博客调用gitment

如gitment项目页Readme所示，在你需要添加评论系统的地方，一般是`_layout/`目录下的 `post.html`, 添加一下代码

```javascript
<div id="container"></div>
<link rel="stylesheet" href="https://jjeejj.github.io/css/gitment.css">
<script src="https://jjeejj.github.io/js/gitment.js"></script>
<script>
var gitment = new Gitment({
  id: '{{ page.date }}', #对应文章评论所形成issue的label，默认为文章的url,为了防止url过长导致评论初始化失败，这里设置label为文章yaml中的时间。这一项无需更改
  owner: 'FrankJKL', #Github Pages博客所在的github账户名
  repo: 'FrankJKL.github.io', #Github Pages博客所在仓库名
  oauth: {
    client_id: 'xxxxxxx', #第1步所申请到的Client ID
    client_secret: 'xxxxxxxxxx',#第1步所申请到的Client Secret
  },
})
gitment.render('container')
</script>
```

填写完上面的内容，把代码保存上传到github就可以了。

## 3、为每篇博文初始化评论系统

由于gitment的原理是为每一篇博文以其YAML中的标题作为标识创建一个github issue， 对该篇博客的评论就是对这个issue的评论。因此，我们需要为每篇博文初始化一下评论系统，初始化后，会在你的github上会创建相对应的issue。

接下来，介绍一下如何初始化评论系统

1. 上面第2步代码添加成功并上传后，你就可以在你的博文页下面看到一个评论框，还有看到以下错误`Error: Comments Not Initialized`，提示该篇博文的评论系统还没初始化

   ![1546944230319](/assert/1546944230319.png)

2. 点击`Login with GitHub`后，使用自己的github账号（必须跟第二步owner用户名相同的账号）登录后，就可以在上面错误信息处看到一个`Initialize Comments`的按钮

   ![](https://jacobpan3g.github.io/img/gitment-in-jekyll.png)

3. 点击`Initialize Comments`按钮后，就可以开始对该篇博文开始评论了， 同时也可以在对应的github仓库看到相应的issue

   ![1546944090342](/assert/1546944090342.png)



## 遇到的一些坑

## Error：NOT FOUND

owner或者repo配置错误了，照着第二步来就好，网页端生成后如下

```javascript
...
<script>
    var gitment = new Gitment({
        id: '{{ page.date }}',
        owner: 'FrankJKL',
        repo: 'FrankJKL.github.io',
        oauth: {
            client_id: 'xxxxxxxx',
            client_secret: 'xxxxxxxxxxxxxxxxxxxxxxxxx',
        },
    })
    gitment.render('container')
</script>
...
```



## Error: Comments Not Initialized

- 在步骤一中，给`Authorization callback URL`指定的地址错了
- 还没有在该页面的Gitment评论区登陆GitHub账号

## Object ProgressEvent

最近gitment作者的服务器过期了，所以登陆GitHub时一直报Object ProgressEvent。我在本文第二步添加的gitment代码是没有这个问题的。

解决：

将`_layout/post.html`中gitment的代码：

```javascript
<link rel="stylesheet" href="//imsun.github.io/gitment/style/default.css">
<script src="//imsun.github.io/gitment/dist/gitment.browser.js"></script>
```

修改为

```javascript
<link rel="stylesheet" href="https://jjeejj.github.io/css/gitment.css">
<script src="https://jjeejj.github.io/js/gitment.js"></script>
```

或

```javascript
<link rel="stylesheet" href="https://jjeejj.github.io/css/gitment.css">
<script src="https://www.wenjunjiang.win/js/gitment.js"></script>
```

登陆时就不会再报错了，这是别人新搭建的服务器。参考https://github.com/jjeejj/jjeejj.github.io/issues/8

## Error：validation failed

发生时间：评论的初始化时

原因：由于gitment的评论是基于GitHub issue的，所以评论初始化时，会生成文章对应的issue，以及通过https://jjeejj.github.io/js/gitment.js生成issue的label。其中的labels有两个，一个是gitment，另一个就是文章的URL，但是label的最大长度限制是50个字符，而我们的URL中包含域名与文章名，有时会超出限制。所以才会发生validation failed

解决：

gitment.js中`labels: labels.concat(['gitment', id])`id默认为`window.location.href`也就是文章的URL，这里的id的作用是每篇文章的评论是要根据id动态加载的，写死的话导致所有的文章共享一个issue。参考前面第二步中将id修改为`'{{ page.date }}'`，就可以覆盖gitment.js中的id。这样传给github issue的label是定长的，不会超过长度限制。同时`date`可以自己写，只要精确到分秒，区分文章不是问题。Good job!

- PS：一个date的例子：`date: 2019-01-07 00:00:00 +0000` 这个date是写到文章顶部的YAML中
- PS：[xjzsq](https://github.com/xjzsq)的方法也很好，思想是用副标题作id，可以看下[他的文章](http://www.xjdesyxx.top/2018/02/07/errsln/)

参考https://github.com/imsun/gitment/issues/118


