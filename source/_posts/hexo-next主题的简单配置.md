---
title: hexo next主题的简单配置
date: 2019-04-07 16:25:56
tags: 
 - 搭建博客
 - 前端
categories: 
 - 前端
   - 搭建博客
---
## <center>Hexo next 简单配置</center>

### 修改next语言

进入站点_config.yml

```
# Site
title: xxx #网站标题
subtitle: xxx #子标题
description:  xxx #描述
keywords:
author: xxx #作者
language: zh-Hans #简体中文
timezone:
```



### 修改主题样式

进入next的_config.yml中，找到下面代码。

```
# ---------------------------------------------------------------
# Scheme Settings
# ---------------------------------------------------------------

# Schemes
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```

用哪个主题就删除#，根据自己的喜好。

---

### 上传头像

```
# Sidebar Avatar
# in theme directory(source/images): /images/avatar.gif
# in site  directory(source/uploads): /uploads/avatar.gif
avatar: /images/head.jpg
```

/images/head.jpg就是头像的路径，把自己的头像放到next/source/images，然后修改路径。

### 开启菜单

about关于，tags标签，categories分类，archives归档，删除#开启

```
menu:
  home: / || home
  #about: /about/ || user
  #tags: /tags/ || tags
  #categories: /categories/ || th
  #archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```

开启之后还需要配置相应的文件夹，在终端定位到站点目录

```
$ hexo new page tags
$ hexo new page categories
```

输入命令后会在source下新建对应的文件夹，然后配置对应的index.md

categories

```
---
title: categories
date: 2019-04-06 22:44:13
type: "categories"
---
```

tags

```
---
title: 标签
date: 2019-04-06 22:44:50
type: "tags"
---
```

---

### 使用菜单

新建一篇文章，然后编辑

```
---
title: "title"
date: 2019-04-07 16:25:56
tags: 
 - first tags
 - second tagis
categories: 
 - xxx
   - xxx
---
```

然后就可以使用这些菜单了。

---
