---
title: Geekpwn SecretCode
date: 2017-09-27 0:0:0
tags:
 - android
---

## Overview
题目给了一个APK, 属于MISC类型, 考察APK发现为存的加密算法分析, 没有JNI, 有Proguard混淆, 需要透过混淆分析Java层代码真实逻辑.

## Analysis

程序的Android源码分为三个部分, 从一张图片读入key, 对key进行处理, 使用用户输入+key得到结果.

```
InputStream v0_1 = this.getResources().openRawResource(R.raw.url);

            int v1 = v0_1.available();
            byte[] v2 = new byte[v1];
            v0_1.read(v2, 0, v1);
            byte[] v0_2 = new byte[16];
            System.arraycopy(v2, 144, v0_2, 0, 16);
            this.para = new String(v0_2, "utf-8");
```
读取图片逻辑, 从便宜144开始读16个字符, 分析得到this_is_the_key.
```
    private String append(String arg4) {
        String v0_2;
        try {
            arg4.getBytes("utf-8");
            StringBuilder v1 = new StringBuilder();
            int v0_1;
            for(v0_1 = 0; v0_1 < arg4.length(); v0_1 += 2) {
                v1.append(arg4.charAt(v0_1 + 1));
                v1.append(arg4.charAt(v0_1));
            }

            v0_2 = v1.toString();
        }
        catch(UnsupportedEncodingException v0) {
            v0.printStackTrace();
            v0_2 = null;
        }

        return v0_2;
    }
```
对key进行处理, 简单的每两位互换位置.
接下来, 由于代码中比较阶段已经知道预期的密文是什么, 用密文加上密钥得到明文.

## Solution

```import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;

public class Solve {


    public static void main(String[] args) throws Exception {
        byte[] output = new byte[]{21, -93, -68, -94, 86, 117, -19, -68,
                -92, 33, 50, 118, 16, 13, 1, -15, -13, 3, 4, 103, -18, 81, 30, 68, 54, -93, 44, -23,
                93, 98, 5, 59};


        String key_str = "this_is_the_key.";
    String key_sft = "htsii__sht_eek.y";

    SecretKeySpec secretKeySpec = new SecretKeySpec(key_sft.getBytes(), "AES");
    Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
    cipher.init(Cipher.DECRYPT_MODE, secretKeySpec);
    String s = new String(cipher.doFinal(output));
    System.out.println(s);
}
}
**flag:LCTF{1t's_rea1ly_an_ea3y_ap4}**
```