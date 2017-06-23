---
title: js-tips
date: 2016-09-23 08:50:52
toc: true
tags:
- js
categories:
- js
---

记录使用js的过程中一些技巧、方法、注意事项。

# js回顾

原文：[js回顾](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/A_re-introduction_to_JavaScript)

# html2pdf方案

1.使用java，itext库：http://developers.itextpdf.com/
2.使用js：
  jsPdf库：https://parall.ax/products/jspdf， https://github.com/MrRio/jsPDF，需要自己拼装PDF中的内容；  
3.使用php，html2pdf库：https://html2pdf.fr/en/download

以下方法先将html转化成图片，再将图片转成pdf
http://www.techumber.com/how-to-convert-html-to-pdf-using-javascript-multipage/
http://www.techumber.com/html-to-pdf-conversion-using-javascript/

`个人比较倾向于使用jsPdf或itext自己拼装PDF内容，这样输出的PDF比较美观，但相对使用起来比较繁琐；使用上述先将HTML转化成图片的方式，输出的图片比较模糊，且无法进行复制等操作；`


