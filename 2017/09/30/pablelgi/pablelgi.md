---
title: pwnable.kr之simple login
date: 2017-09-30 11:33:22
tags: 
 - pwn
 - stack
---

这道题主要考察栈溢出. 在进入正题之前, 首先还是对栈帧结构做一个总结, 因为之前很久没做都忘了.

![](1.PNG)

如图所示, 我总结的栈结构和其他说法有些不同, 方便自己记忆吧, 是否合理有待讨论.

在这里我认为函数调用是父函数中断在某一个位置, 然后开始执行子函数的逻辑. EBP通常翻译成基址寄存器, 也就是当前栈底, 问题是父函数也有父函数, 而寄存器EBP只有一个, 因此在函数跳转时, 需要把父父函数的EBP保存在栈上. 另外需要保存的是父函数中断点PC后的下一个位置, 也就是当子函数执行回来时候, 父函数需要继续执行的位置, 很多材料里这个地址叫做ret, 感觉容易和汇编指令搞混.

接下来上(IDA F5)代码

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char *decoded; // [esp+18h] [ebp-28h]
  char s[30]; // [esp+1Eh] [ebp-22h]
  unsigned int len; // [esp+3Ch] [ebp-4h]

  memset(s, 0, 30u);
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  printf("Authenticate : ");
  _isoc99_scanf("%30s", s);
  memset(&secret, 0, 12u);
  decoded = 0;
  len = Base64Decode((int)s, (char *)&decoded);
  if ( len > 12 )
  {
    puts("Wrong Length");
  }
  else
  {
    memcpy(&secret, decoded, len);
    if ( auth(len) == 1 )
      correct();
  }
  return 0;
}
```

```c
_BOOL4 __cdecl auth(int len)
{
  char v2; // [esp+14h] [ebp-14h]
  char *s2; // [esp+1Ch] [ebp-Ch]
  char *local_secret; // [esp+20h] [ebp-8h]

  memcpy(&local_secret, &secret, len);
  s2 = (char *)calc_md5((int)&v2, 12);
  printf("hash : %s\n", s2);
  return strcmp("f87cd601aa7fedca99018a8be88eda34", s2) == 0;
}
```

```c
void __noreturn correct()
{
  if ( secret == 0xDEADBEEF )
  {
    puts("Congratulation! you are good!");
    system("/bin/sh");
  }
  exit(0);
}
```

程序给了带shell的correct函数, 因此不需要构造shell, 一开始看到程序是静态编译, 因为要用ROP, 胡思乱想了半天.

按照正常的逻辑执行, 用户输入正确的密码之后, auth函数进行校验通过, 条用correct函数, 验证secret前4字节=0xdeadbeef, 直接给出shell. 问题是, 这个正确密码无从得知, 需要通过溢出绕过验证.

大概是为了方便构造用户输入, 程序自带了贴心的base64, 脱去这层外衣之后, 实际受控的用户输入只有12个, 很不够用. 而且F5对程序的分析不理想, 一开始没有找到漏洞点. 事实上, 漏洞点在auth函数的汇编中可以看到, 很隐晦.



![](2.PNG)

此处memcpy参数len最多12, src是secret, dest是eax中的指针, eax = [ebp + var_14] + 0xc, 注意var_14是ebp-14h, 那么eax最终指向ebp-14h+8h=ebp - 8, 就是图中的anonymous_0位置.

![](3.PNG)

这个位置最多接收12个字节, 但是实际上当前函数只有8个字节, 多余四个会溢出并覆盖父函数EBP. 想要覆盖父函数EIP不够.

父函数EBP效果不如父函数EIP, 后者可以直接在返回时控制指令流. 但是控制父函数EBP也够了, 如前面论述, 父函数EBP指向父函数的栈底, 与父函数的返回值距离很近. 我们不能控制auth的返回值, 它一定返回到main, 但是main的栈底可以控制, 而栈底下一个元素就是main的返回值.

接下来寻找可控的"伪栈帧"位置, 前面提到,12位的secret,最后四位是溢出点, 前面8位可以布置成栈帧状:前四位是父父函数的EBP, 这里为了过检测必须是0xdeadbeef, 中间四位是correct地址. 这样一来, 可以构造输入如下:

`base64.base64encode(p32(0xdeadbeef) + p32(addr_correct) + p32(addr_secret))`

即可拿到flag.