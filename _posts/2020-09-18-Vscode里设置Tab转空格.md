---
layout:     post
title:      VS Code里设置tab转空格 
subtitle:   怎么完美结合空格和tab的优点
date:       2020-09-18
author:     mengxun
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - VS Code
---

### header

用空格还是tab表示缩进一直都存在争议:)但是最近在vsocde里发现可以完美都结合的结合二者

### body

先说说各自优缺点
- tab的优点是方便，摁一下键盘就够了；缺点是不同的环境，tabsize不一定一样，有的是length = 4 spaces，有的是等于 8 spaces，可能在你的环境缩进正常，到了同事环境就乱掉了
- 空格的优点是：空格就是空格，到哪都不会变，缩进不会乱掉；缺点是累。。

其他编辑器或者ide能不能设置我不太清楚（应该也可以），但是在vsocde可以通过设置去结合他们——缩进的时候只需要摁一次tab即可，然后vsocde会帮你转成四个空格
这样的话既不累，空格也有了，也不用担心缩进会乱掉


如果要配置的话主要是配置 insertSpaces 属性，这个属性意思：按tab的时候不插入tab，而是用空格取代；至于具体的看stackoverflow吧

参考链接：https://stackoverflow.com/questions/32720854/whats-the-difference-between-editor-insertspaces-and-editor-tabsize-in-sett

