---
title: js-trouble-shooting
date: 2016-08-09 18:32:04
toc: true
tags:
- js
categories:
- js
---
记录使用js时遇到的问题、原因及解决办法。

# chrome display不生效

## 问题描述

往后台请求数据时有一定的延时，为了更好的用户体验，需要在页面进行查询时，先展示一个loading的gif图片。
关键代码如下：
```
$("#id").css("display", "none");
$("#id").css("display", "block");
```
在firefox下正常显示，在chrome下不能正常展示，而在chrome下进行调试时，在此处打上断点，便又可正常显示。

## 原因
在js中使用ajax向后台请求数据时，采用了同步的方式`async: false`引起。
一点猜测：由于使用`async: false`时，Ajax请求会将整个浏览器锁死，而在chrome下`$("#id").css("display", "none");`尚未渲染完成，因此无法看到加载图片。

## 解决办法
使用ajax请求默认的异步请求方式，注释掉`async: false`或将其改为`async: true`。



