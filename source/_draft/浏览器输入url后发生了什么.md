---
title: 浏览器输入url后发生了什么
date: 2021-02-19 12:33:48
tags:
---

这是一个老生常谈的问题了，网上也有很多文章，但是让自己说却说不出个一二三来，因此写下来加深对这个过程的理解，并且能将计算机网络的知识串联起来，不再是单独的一个个知识点。

<!--more-->

# DNS解析
互联网是通过ip进行联系的，而不是域名，因此第一步需要根据域名拿到对应的ip。

1. 浏览器会检查是否有该域名的dns缓存
2. 缓存中没有，在hosts文件中查找
3. hosts文件中也没有，向dns服务器发起请求

## DNS解析过程

以请求`github.com`为例：
1. dns客户端向dns解析器发起解析请求
2. dns解析器会选择距离较近的跟域名服务器，请求顶级域名dns服务器地址
3. 拿到顶级域名`com.`dns服务器地址后，再向其请求`github.com.`的命名服务
4. 得到授权的dns服务后，就可以根据具体的host记录查找对应的ip了


