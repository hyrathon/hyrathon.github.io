---
title: 2016武汉某宣传周题目----Hurry Up
date: 2017-09-27 0:0:0
tags:
 - android
---

## Overview

题目给的APK包含Java和Android层代码, 代码结构清晰没有太多混淆, 反调试等.

## Analysis

程序的入口MainActivity有问题, 无任何实际代码, 分析发现实际有效的是GetActivity, 调用了Encryption类和native的一些方法. 尝试改包或者`am start`强行启动GetActivity均告失败.分析so库发现, 代码的第一次验证函数getsign对整包签名做了native层校验, 且结果加入了后续flag计算.

写了一个小工程GetSign, 通过PackageManager获取了程序的签名, 很长一千多个字节, 将其传给第二个native函数getAnswer以期绕过完整性检验.
```
PackageManager  pm = getPackageManager();
        try {
            PackageInfo pi =  pm.getPackageInfo("com.example.root.crack_me2", PackageManager.GET_SIGNATURES);
            for (Signature s : pi.signatures){
                ssss += s.toCharsString() + "!!!!!!!!!!!!";
            }
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
            ssss += "error";
        }

```

结果程序无限卡死, 无法通过建立Activity的onCreate函数, 分析native函数发现其中计算返回值的算法时间复杂度极高, 有限时间内根本运算不完(下面列了出来), 但是, 其中有`mod 45454`的运算, 猜测结果不会超过这个范围, 因此, 可以暴力破解.

```
signed int __fastcall Java_com_example_root_crack_1me2_GetActivity_getAnswer(JNIEnv *env, jobject jobj, int int_50213)
{
  signed int result; // r0@1
  int v4; // r1@1
  int v5; // r7@1
  signed int v6; // r5@1
  signed int v7; // r4@2
  signed int v8; // r6@3
  signed int v9; // r1@4

  j_j___aeabi_idivmod(int_50213, 45454);
  result = 1;
  v5 = v4;
  v6 = 2110101010;
  do
  {
    v7 = 2110101010;
    do
    {
      v8 = 2110101010;
      do
      {
        j_j___aeabi_idivmod(result * v5, 45454);
        --v8;
        result = v9;
      }
      while ( v8 );
      --v7;
    }
    while ( v7 );
    --v6;
  }
  while ( v6 );
  return result;
}
```

## Solution

写出来的暴力程序如下, 由于理论上有45454种结果, 而结果参与了后续运算, 不是所有的结果参与Encryption的运算都能返回结果, 另外, flag等字样很有可能出现在最终结果中, 因此可以进一步筛选.

```
        String v3 = null;
        String v1 = "tQJWYl+8C4cO2jwq782P5qc/9FXVF6cedVTtozSnyFk=";
        for (int i = 0; i < 45454; ++i) {
            try {
                v3 = Encryption.decrypt(Integer.toString(i), v1);
            } catch (Exception e) {

            }
            if (v3 == null)
                continue;
            if (v3.contains("flag")) {
                textView.setText(v3);
                v3 = null;
            }
        }
```

结果是getAnswer函数返回9953, 对应flag为`flag{Y0u_need_t0_hu55y_up}`.