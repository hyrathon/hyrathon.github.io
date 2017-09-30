---
title: SECCON Quals CTF 2015 Reverse-Engineering Android APK 1
date: 2017-09-28 16:49:43
tags: 
 - android
---

# SECCON Quals CTF 2015: Reverse-Engineering Android APK 1

[Github Link](https://github.com/ctfs/write-ups-2015/tree/master/seccon-quals-ctf-2015/binary/reverse-engineering-android-apk-1)

> 题目比较简单,不多赘述


## Overview
不需要Overview,直接看代码逻辑即可

## Analyse
根据惯例对apk反编译,直接看Java代码
```Java
if (1000 == MainActivity.this.cnt)
    localTextView.setText("SECCON{" + String.valueOf(107 * (MainActivity.this.cnt + MainActivity.this.calc())) + "}");
```
可以看出flag是形如`SECCON{107*(1000 + calc)}`形式,只有一个未知量是函数calc返回值,而这个函数是native函数,将lib中任意一个so文件拖入IDA,找到对应函数形式如下所示
```
public Java_com_example_seccon2015_rock_1paper_1scissors_MainActivity_calc
Java_com_example_seccon2015_rock_1paper_1scissors_MainActivity_calc proc near
mov     eax, 7
retn
Java_com_example_seccon2015_rock_1paper_1scissors_MainActivity_calc endp

_text ends
```
竟然直接返回7!
## Solve
`107*1007=107749`
flag is SECCON{107749}






