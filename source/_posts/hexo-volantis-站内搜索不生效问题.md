---
title: hexo + volantis 站内搜索不生效问题
author: Gaesar
date: 2026-10-09 00:06:59
tags: bug复盘
categories:
---

# hexo + volantis 站内搜索不生效问题

反复尝试了\_config.yml 和\_config.volantis.yml的各种配置，结果一直不生效，甚至尝试了给volantis添加js源码，还是不生效。

最后把\_config.yml 和\_config.volantis.yml中的所有配置都删除了，结果生效了......

复盘发现，最开始不生效，是因为没有安装

```
+-- hexo-generator-json-content@4.2.3
+-- hexo-generator-search@2.4.3
```

这两个插件安装好，不用做任何配置，走默认配置，站内搜索就生效了。

