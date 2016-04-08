---
title: 欢迎
date: 2016/04/06 00:00:00
toc: true
tags: 
- hexo
categories: 
- hexo
---
基于hexo搭好了个人博客，在这里记录并测试一下。

# hexo的安装步骤
参考官方文档：[hexo.io](https://hexo.io/zh-cn/docs/index.html)
# 注意事项
## Node.js版本
* 不使用最新版本的Node.js，使用官方文档建议的v4；

## disqus集成
* 注册disqus；
* 将short_name添加到_config.yml；
* 注意修改disqus设置为中文；
* 不使用评论：
	在front-matter中添加一行
	```
	comments: false
	```

## maupassant主题
使用[屠城](https://www.haomwei.com/)的主题，感谢！
* 添加关于(about)，历史（history），留言（guestbook）
```
hexo new page abou
hexo new page history
hexo new page guestbook
```

* _config.yml的menu中添加：
```
- page: about
  directory: about/
  icon: fa-user
- page: history
  directory: history/
  icon: fa-book
- page: guestbook
  directory: guestbook/
  icon: fa-comments
```
* 添加时间轴
```
---
title: 时间轴
date: 2016-04-07 15:12:10
comments: false
---
<div class="timeline">
	<div class="timeline-item active">
		<div class="year">2014<span class="marker"><span class="dot"></span></span></div>
		<div class="info">
			注册域名jackalope.cn
		</div>
	</div>
	<div class="timeline-item active">
		<div class="year">2016<span class="marker"><span class="dot"></span></span></div>
		<div class="info">初试个人博客，使用hexo搭建，托管于Github</div>
	</div>
</div>
```

## 配置文件
{% asset_path _config.yml 全局配置 %}
{% asset_path maupassant_config.yml 主题配置 %}

# Mou的使用

## 标题
将#号写在行首表示标题，标题1 - 标题6，数字对应#号个数

## 加粗
未加粗/**加粗**（aaa/**aaa**）(快捷链：Cmd + B)

## 倾斜
未倾斜/*倾斜*（aaa/*aaa*）(快捷键：Cmd + I)

## 引用
> 把 > 写到开头，本质上是生成blockquote标签

## 链接
[jackalope](http://jackalope.cn)

## 邮件
<zowie_chen@163.com>

## 图片
内嵌图片，直接写入具体图片地址
![内嵌图片](http://25.io/smaller/favicon.ico "Title here")

引用图片，先指定图片名称，之后再将图片名称映射到具体图片地址
![引用图片][图片名称]
[图片名称]: http://resizesafari.com/favicon.ico "Title"

## 代码块
使用3个及以上着重符(`)开始并结束：
```
System.out.println("Hello world!");
```

## 有序列表
使用"1." + 空格
1. 111
2. 222
3. 333

## 无序列表
使用"*" + 空格
* 111
* 222
* 333

或者"-" + 空格
- 111
- 222
- 333

## 换行符
两个及以上空格即可生成一个换行符`<br />`

## 水平分隔符
三个及以上 "*" 或 "-" 或 "- "，每行之前有空行

***

---

- - -

## 脚注
脚本，跳到标题[^1]
[^1]: #标题

## 删除线
使用两个~开始并结束
~~ZZZZZZzzzzzz~~

## 表格
列使用|分开，头尾的|可加可不加；
列名与行使用多个-分开，:----表示左对齐，:----:表示两端对齐，----:表示右对齐，`PS: 表格上方有一空行`

列1|列2|列3
--|--|--
11|12|13
21|22|23
31|32|33


`列补齐"|"`

|列1 | 列2 | 列3|
|----|----|----|
|11  |12  |13|
|21  |22  |23|
|31  |32  |33|


`左中右对齐`

列1|列2|列3
:--|:--:|--:
11|12|13
21|22|23
31|32|33

## 快捷链
### 视图
* 预览: Shift + Cmd + I
* 统计字数: Shift + Cmd + W
* 透明: Shift + Cmd + T
* 悬浮，永在最前: Shift + Cmd + F
* 左右窗口大小1/1: Cmd + 0
* 左右窗口大小3/1: Cmd + +
* 左右窗口大小1/3: Cmd + -
* 书写方向: Cmd + L
* 全屏/退出全屏: Control + Cmd + F

### 操作
* 拷贝html: Option + Cmd + C
* 加黑加粗: Select text, Cmd + B
* 倾斜: Select text, Cmd + I
* Inline Code: Select text, Cmd + K
* Strikethrough: Select text, Cmd + U
* Link: Select text, Control + Shift + L
* Image: Select text, Control + Shift + I
* Select Word: Control + Option + W
* Select Line: Shift + Cmd + L
* Select All: Cmd + A
* Deselect All: Cmd + D
* Convert to Uppercase: Select text, Control + U
* Convert to Lowercase: Select text, Control + Shift + U
* Convert to Titlecase: Select text, Control + Option + U
* Convert to List: Select lines, Control + L
* Convert to Blockquote: Select lines, Control + Q
* Convert to H1: Cmd + 1
* Convert to H2: Cmd + 2
* Convert to H3: Cmd + 3
* Convert to H4: Cmd + 4
* Convert to H5: Cmd + 5
* Convert to H6: Cmd + 6
* Convert Spaces to Tabs: Control + [
* Convert Tabs to Spaces: Control + ]
* Insert Current Date: Control + Shift + 1
* Insert Current Time: Control + Shift + 2
* Insert entity <: Control + Shift + ,
* Insert entity >: Control + Shift + .
* Insert entity &: Control + Shift + 7
* Insert entity Space: Control + Shift + Space
* Insert Scriptogr.am Header: Control + Shift + G
* Shift Line Left: Select lines, Cmd + [
* Shift Line Right: Select lines, Cmd + ]
* New Line: Cmd + Return
* Comment: Cmd + /
* Hard Linebreak: Control + Return

### 编辑

* Auto complete current word: Esc
* Find: Cmd + F
* Close find bar: Esc

### Post

* Post on Scriptogr.am: Control + Shift + S
* Post on Tumblr: Control + Shift + T

### 导出

* Export HTML: Option + Cmd + E
* Export PDF:  Option + Cmd + P


### And more?

Don't forget to check Preferences, lots of useful options are there.

Follow [@Mou](https://twitter.com/mou) on Twitter for the latest news.

For feedback, use the menu `Help` - `Send Feedback`

# hello-world.md

