---
layout: post
title:  "《Linux/Unix设计思想》"
date:   2016-05-13 22:17:06 +0000
categories: 读书笔记 
---
SMALL 小即是美

1THING 让每个程序只做好意见事情

PROTO 尽快建立原型

PORT 舍高效率而取可移植性

FLAT 使用纯文本文件来存储数据

REUSE 充分利用软件的杠杆效应

SCRIPT 使用shell脚本来提高杠杆效应和可移植性

NOCUI 避免强制性的用户界面

FILTER 让每一个程序成为过滤器

custom 允许用户定制环境

kernel 尽量使操作系统的内核小而轻巧

lcase 使用小写字母并尽量简短

trees 保护树木

silence 沉默是金

parallel 并行思考

sum 各部分之和大于整体

90cent 寻找90%的解决方案

worse 更坏就是更好

hier 层次化思考



{% for category in site.categories %}
<h3>{{ category | first }} ({{ category | last | size }})</h3> 
<ul class="arc-list">
{% for post in category.last %} 
<li>{{ post.date | date:"%d/%m/%Y"}}<a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul> 
{% endfor %}
