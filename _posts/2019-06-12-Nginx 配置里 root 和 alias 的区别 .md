---
layout:     post
title:      Nginx 配置里 root 和 alias 的区别
subtitle:   root 和 alias 指令的差异
date:       2019-06-12
author:     mengxun
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Nginx
---

#### root 的写法

```
location /html/ {
	root /home/data/;
}
```

当一个 URL 的路径部分为 `/html/index.html`的时候，则最终请求的资源的路径相当于`/home/data/html/index.html`

#### alias 的写法

```
location /html/ {
	alias /home/data/;
}
```

当一个 URL 的路径部分为 `/html/index.html`的时候，则最终请求的资源的路径相当于`/home/data/index.html`

#### 两者的区别

第一种写法的最终路径等于 root 指令指定的路径加上 url 的完整路径；而第二种写法的话，nginx则会用alias指定的路径去替换url里的部分内容，替换掉的这部分内容就是location，类似 path = url.replace(location, alias)。



