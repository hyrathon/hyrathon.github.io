---
title: 360 Hello CTF
date: 2017-09-28 16:49:43
tags: 
 - android
---

# HelloCTF

纯Java,没有任何反调试,实质上是一道编程题.

逐行仔细分析代码可以不断得到关系,输入是一个22字符长的小写a-z字符串,约束关系总结如下:

v8 v9 = 4   19 分别是上述序号英文字符

四次出现的位置5 8 10 12

三次 3 16 19



![1](helctf/1.png)



 

pos4 ^ pos3 = 22

pos6^pos7=5

pos7^pos8 = 12

 

p9=118

p11=109

p13 = 106

p14 117

p15 115

(ord



 

有一个watch夹在中间

以don开头

得到结论dontbelievemejustwatch