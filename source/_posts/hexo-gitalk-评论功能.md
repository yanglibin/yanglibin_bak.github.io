---
title: hexo gitalk 评论功能
date: 2021-09-05 09:54:10
tags:
- hexo
- gitalk

---



# 1. 介绍

hexo 中常用的评论工具有如下几种:

- Valine(评论不需要账号，数据托管于Leancloud数据库管理)
- Gitalk(可以使用，评论需要登录GitHub账号)
- 畅言(搜狐的评论系统,很强大,但需要有备案号)
- Gitment(登录GitHub账号长时间loading)
- 来必力(注册受阻，且是韩国服务器，评论系统响应变慢)
- 友言(不需要备案号,功能也比较强大)
- disqus(国内经常被墙)
- 网易云跟帖(已经关闭系统)
- 多说(功能强大，但是已经停止服务)



> Gitalk 是一个基于 GitHub Issue 和 Preact 开发的评论插件。
<!-- more -->

[gitalk项目](https://github.com/gitalk/gitalk/blob/master/readme-cn.md)



有如下特性:

- 使用 GitHub 登录
- 支持多语言 [en, zh-CN, zh-TW, es-ES, fr, ru, de, pl, ko, fa, ja]
- 支持个人或组织
- 无干扰模式（设置 distractionFreeMode 为 true 开启）
- 快捷键提交评论 （cmd|ctrl + enter）

# 2. 部署

## 2.1. 创建 `gitalk.ejs` 

> 在你的 hexo 目录 `themes/yilia/layout/_partial/post/` 目录下创建 `gitalk.ejs` 并写入如下内容：

```js
<div id="gitalk-container"></div>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.css">
<script src="https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.min.js"></script>
<script src="https://cdn.bootcss.com/blueimp-md5/2.10.0/js/md5.js"></script>

<script>
var gitalk = new Gitalk({
  clientID: '<%=theme.gitalk.clientID%>',
  clientSecret: '<%=theme.gitalk.clientSecret%>',
  repo: '<%=theme.gitalk.repo%>',
  owner: '<%=theme.gitalk.owner%>',
  admin: ['<%=theme.gitalk.admin%>'],
  id: md5(window.location.pathname),
  distractionFreeMode: <%=theme.gitalk.distractionFreeMode%>
})

gitalk.render('gitalk-container')
</script>

```

## 2.2. 修改 article.ejs

> 在你的 hexo 目录`themes/yilia/layout/_partial/article.ejs` 文件中最后一行 `<% } %>` 之前添加如下内容：

```js
<% if(theme.gitalk.enable && theme.gitalk.distractionFreeMode){ %>
      <%- partial('post/gitalk', {
      key: post.slug,
      title: post.title,
      url: config.url+url_for(post.path)
    }) %>
  <% } %>
```

## 2.3. 添加配置文件

> 在 yilia 的配置文件_config.yml 中 gitment 配置下面添加如下配置文件

```yaml
#6. Gitalk
# Gitalk
gitalk: 
  enable: true    
  clientID: 'your clientID'    #Github上生成的
  clientSecret: 'your clientSecret'   #同上
  repo: blog-talk   #评论所在的github project
  owner: yanglibin  #github用户名
  admin: yanglibin  #可以初始化评论issue的github账户名称
  distractionFreeMode: true
```

## 2.4. 注册 OAuth Application

> 当别人评论你的文章时，会需要它是授权。

[注册地址](https://github.com/settings/applications/new)

`Homepage URL`, `Authorization callback URL` 写你的blog 地址, 其它的看心情写。

## 2.5. 创建存放评论的仓库

在 github 上仓库一个公开的仓库，用来存放评论。
这里使用的是`blog-talk`



# 3 参考: 

- [为Hexo yilia主题添加gitalk评论](https://www.dazhuanlan.com/isnowfox/topics/1401771)
