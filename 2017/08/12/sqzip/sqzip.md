---
title: 神奇的zip 顺藤摸瓜
date: 2017-09-27 0:0:0
tags:
 - android
---## 神奇的zip

首先拿到的apk坏了, 拖到linux下`zip -ff testndk4_Signed4.zip`修复一下, 第一个知识点完成.

题目是纯的Java加上存的JNI, Java下没有需要分析的逻辑.

首先需要绕过一处一定会退出的检查, 修改so文件里的move r0,0为move r1,1强行返回1. 或者直接动态调在此处下断点改寄存器过去.
```
        super.onCreate(arg5);
        this.setContentView(2130903065);
        if(this.isExit()) {
            this.startActivity(new Intent(((Context)this), MainActivity.class));
        }
        else {
            Timer v0 = new Timer();
            b v1 = new b(this);
            Toast.makeText(((Context)this), "抱歉，请先获得权限，再进入！！", 0).show();
            v0.schedule(((TimerTask)v1), 5000);
        }
```

题目有个问题对动态调试没有做任何限制,因此在最终需要比较的密码处下个断点看内存即可发现flag.
```
int __fastcall Java_com_example_testndk4_MainActivity_encodePassword(int a1)
{
  int v1; // r5@1
  const char *v2; // r7@1
  char *src; // ST04_4@1
  char *v4; // ST04_4@1
  char *v5; // r0@2
  const char *v6; // r1@2
  int result; // r0@4
  char v8; // [sp+8h] [bp-58h]@1
  char v9; // [sp+14h] [bp-4Ch]@1
  char s; // [sp+28h] [bp-38h]@1
  int v11; // [sp+44h] [bp-1Ch]@1

  v1 = a1;
  v11 = _stack_chk_guard;
  v2 = (const char *)Jstring2CStr();
  j_j_memcpy(&v9, "thinkingInAndroid", 0x12u);
  src = (char *)encodePS(&v9);
  j_j_memset(&s, 0, 0x1Au);
  j_j_strcpy(&s, src);
  v4 = (char *)encodePS(&s);
  j_j_memset(&v8, 0, 0xAu);
  if ( j_j_strcmp(v2, v4) )
  {
    v5 = &v8;
    v6 = "Sorry!";
  }
  else
  {
    v5 = &v8;
    v6 = "Sucess!";
  }
  j_j_strcpy(v5, v6);
  result = (*(int (__fastcall **)(int, char *))(*(_DWORD *)v1 + 668))(v1, &v8);
  if ( v11 != _stack_chk_guard )
    j_j___stack_chk_fail(result);
  return result;
}
```
看源码或者F5都可以发现程序是对`thinkingInAndroid`字符串做了复杂的运算,但是用户输入没有参与进来,最后将用户输入与复杂预算的结果比较, 直接在那里下断点输出:
```
MOVS    R0, R7          ; s1
LDR     R1, [SP,#0x60+src] ; s2
--> set bp here :BL      j_j_strcmp
CMP     R0, #0
BNE     loc_1042
```

定位到内存,得到flag:lxienietIeAehfyih

## 顺藤摸瓜

单纯的jni逻辑分析, java代码只涉及一点.
```
int __fastcall Java_com_example_demo_MainActivity_check(int a1, int a2, int a3)
{
  int v3; // r4@1
  int v4; // r6@1
  int v5; // r7@1
  void *v6; // ST08_4@1
  int v7; // r0@1
  int v8; // r7@1
  signed int v9; // r6@1
  void *v10; // r5@2
  signed int v11; // r0@4
  signed int i; // r3@4
  int result; // r0@7
  void *src; // [sp+8h] [bp-2F0h]@1
  char dest[56]; // [sp+14h] [bp-2E4h]@4
  char v16[200]; // [sp+4Ch] [bp-2ACh]@4
  char v17; // [sp+114h] [bp-1E4h]@4
  char s; // [sp+1DCh] [bp-11Ch]@4
  int v19; // [sp+2DCh] [bp-1Ch]@1

  v3 = a1;
  v4 = a3;
  v19 = _stack_chk_guard;
  v5 = (*(int (**)(void))(*(_DWORD *)a1 + 24))();
  v6 = (void *)(*(int (__fastcall **)(int, const char *))(*(_DWORD *)v3 + 668))(v3, "GB2312");
  v7 = (*(int (__fastcall **)(int, int, const char *, const char *))(*(_DWORD *)v3 + 132))(
         v3,
         v5,
         "getBytes",
         "(Ljava/lang/String;)[B");
  v8 = (*(int (__fastcall **)(int, int, int, void *))(*(_DWORD *)v3 + 136))(v3, v4, v7, v6);
  v9 = (*(int (__fastcall **)(int, int))(*(_DWORD *)v3 + 684))(v3, v8);
  src = (void *)(*(int (__fastcall **)(int, int, _DWORD))(*(_DWORD *)v3 + 736))(v3, v8, 0);
  if ( v9 <= 0 )
  {
    v10 = 0;
  }
  else
  {
    v10 = j_j_malloc(v9 + 1);
    j_j_memcpy(v10, src, v9);
    *((_BYTE *)v10 + v9) = 0;
  }
  (*(void (__fastcall **)(int, int, void *, _DWORD))(*(_DWORD *)v3 + 768))(v3, v8, src, 0);
  j_j_memset(v16, 0, 0xC8u);
  j_j_memset(&v17, 0, 0xC8u);
  j_j_memset(&s, 0, 0x100u);
  j_j_memcpy(dest, &unk_2454, 0x38u);
  v11 = j_j_strlen((const char *)v10);
  for ( i = 0; i < v11; ++i )
    v16[i] = *((_BYTE *)v10 + i) + 97 - dest[4 * i];
  v16[v11 & (~v11 >> 31)] = 0;
  n1(v16, "nbrcdpassword", &v17);
  n2(&v17, &s);
  result = (unsigned int)j_j_strcmp(&s, "7405847394833303439294822334") <= 0;
  if ( v19 != _stack_chk_guard )
    j_j___stack_chk_fail(result);
  return result;
}
```
先贴一下F5, 其中没有用的逻辑很多, 基本就是如何如何从java拷贝到c申请内存然后到处挪.ida改的变量名没保存.变量里边`dest`有点用从内存一遍读取了一段加密向量. 然而这些都没有卵用.......
```
# 我们白白做了这么多工作
#!/bin/env python
#coding:utf-8
import string

iv1 = [0x3F,0x4D,0x6C,0x5B,0x54,0x5B,0x6C,0x5B,0x54,0x46,0x38,0x46,0x3F,0x1C]
iv2 = "nbrcdpassword"
#iv2_alpha = [ord(x) - ord('a') for x in iv2]


final = "7405847394833303439294822334"
#byte1 = [int(final[x:x+2:][::-1]) + 72 - 97 for x in range(0, len(final), 2)]
byte1 = map(int, list(final))
print byte1

#print string.printable
for i in range(14):
	for k in "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_":
		k1 = ord(k) - iv1[i] + 0x61 - ord('a')
		k2 = (k1 + ord(iv2[i % len(iv2)]) - ord('a'))%26 + 97 - 72
		if k2 % 10 == byte1[2*i] and (k2/10)%10 == byte1[2*i+1]:
			print k,
	print 
	
```
因为我们可以从java逻辑中发现jni做不做只影响判断, 对输出没有影响......
```
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.requestWindowFeature(1);
        this.setContentView(2130903065);
        this.D_text = this.findViewById(2131296319);
        String v0 = this.D_text.getText().toString();
        String v3 = "";
        int v2;
        for(v2 = 0; v2 < v0.length(); ++v2) {
            try {
                v0.charAt(v2);
                v3 = String.valueOf(v3) + (((char)(v0.charAt(v2) - 8)));
            }
            catch(Exception v1) {
                v3 = String.valueOf(v3) + v0.charAt(v2);
            }
        }

        this.D_text.setText(((CharSequence)v3));
    }
}
```
上面的代码, 当jni执行了或者被绕过之后, 逻辑是对一段R中的烫烫烫进行了移位.
`碸夼圸烁Ｂ夲皅卟跷:叿 + 8 = 西电的校址会面 = flag`