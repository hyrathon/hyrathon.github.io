---
title: 南京CTF2017 easycrack
date: 2017-09-28 16:49:43
tags: 
 - android
---


**Category:** Mobile
**Points:** 100
**Solves:**43
**Description:** None

## Write-up

ARM二进制分析题, Java里边基本没有内容, 只有一个native需要的固定值msg是从Java中计算然后传进去的, 写个脚本跑一下就能得到这个值.

程序运行流程是经典的字符串加密, 用户输入经过与msg的异或处理后被与一个I_am_the_key字符串经过复杂但是固定的运算流程得到的值进行加密, 我们称这个值为buffer, 加密过程 buffer 与处理过的用户输入进行加密, 但是加密算法实质是异或可逆, 因此在得知结果的时候即可算出用户输入. 叙述不是特别明确, 简单画图解释:

user_input(?)  ------> xor  ----> 处理过的输入------------------------------------>crypt-------------->结果

msg-------------------->              I_am_the_key------>process------>buffer--->

通过逆向发现结果作为明文字符串存储在二进制文件中, 因为上述每一步都可逆, 因此可以编写脚本反过来执行, 从结果获得最初的用户输入. 分析过程比较简单, 不具体说明, 详见idb

```python
debug = False

#seems like a char shift
def init(s="I_am_the_key"):
    buffer1 = []
    buffer2 = ""
    for i in range(256):
        buffer1 += chr(i)
        buffer2 += s[i % len(s)]
    if debug :
        print buffer1
        print buffer2

    v7 = 0
    for j in range(256):
        v9 = buffer1[j]
        v7 = (v7 + ord(v9) + ord(buffer2[j])) % 256
        buffer1[j] = buffer1[v7]
        buffer1[v7] = v9
    return map(ord, buffer1)

def getcompare():
    cmpstr = "C8E4EF0E4DCCA683088134F8635E970EEAD9E277F314869F7EF5198A2AA4"
    compare = []
    for i in range(len(cmpstr)/2):
        compare += cmpstr[2 * i:2 * i+2].decode('hex')
    return map(ord, compare)

def decrypt(buffer, compare):
    v3=0
    v4=0
    for i in range(30):
        v3 = (v3 + 1) % 256;
        v5 = buffer[v3];
        v4 = (v5 + v4) % 256;
        buffer[v3] = buffer[v4];
        buffer[v4] = v5;
        compare[i] ^= buffer[(buffer[v3] + v5) % 256];
    return compare

def getuserinput(beforecrypt):
    msg = map(ord, "V7D=^,M.E")
    for i in range(30):
        beforecrypt[i] ^= msg[i % 9]
    return "".join(map(chr, beforecrypt))


buffer = init()
compare = getcompare()
beforecrypt = decrypt(buffer, compare)
print getuserinput(beforecrypt)
```

