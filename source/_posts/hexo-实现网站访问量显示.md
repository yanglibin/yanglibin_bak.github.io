---
layout: hexo
title: hexo网站访问量显示
date: 2021-09-05 09:52:54
tags:
- hexo
- 不蒜子
---

# "不蒜子"实现hexo网站访问量显示

[不蒜子官方链接](http://ibruce.info/2015/04/04/busuanzi/)


`themes/yilia/layout/_partial/footer.ejs` 文件的`footer` 标签下加如下内容:


```js
<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>

本站总访问量<span id="busuanzi_value_site_pv"></span>次

本站访客数<span id="busuanzi_value_site_uv"></span>人次 

本文总阅读量<span id="busuanzi_value_page_pv"></span>次

```
