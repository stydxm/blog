---
title: 给Hexo添加脚注支持
date: 2022-09-24 00:32:22
categories: 技术
tags: 博客
---

hexo有现成的不错的方案[`markdown-it-footnote`](https://github.com/markdown-it/markdown-it-footnote)可用，所以也不用怎么折腾，但是有几个坑

<!--more-->

markdown-it-footnote的[README](https://www.npmjs.com/package/markdown-it-footnote)写着
> ## Install
> node.js, browser:
> ```bash
> npm install markdown-it-footnote --save
> bower install markdown-it-footnote --save
> ```

但并不是装上这个库就能用脚注了 ~~（踩坑了）~~

首先[`markdown-it-footnote`](https://github.com/markdown-it/markdown-it-footnote)是[Markdown it](hhttps://markdown-it.github.io/)的一个插件，所以先把hexo默认的[`hexo-renderer-marked`](https://github.com/hexojs/hexo-renderer-marked)渲染引擎替换成[`hexo-renderer-markdown-it`](https://github.com/hexojs/hexo-renderer-markdown-it)

```bash
npm un hexo-renderer-marked --save
npm i hexo-renderer-markdown-it --save
```
这一步挺常规的，然后就是插件需要手动启用 ~~没仔细看readme~~  
在配置文件`_config.yml`里加入`markdown`这个字段，并启用脚注插件

```yaml
markdown:
  plugins:
    - markdown-it-footnote
```
