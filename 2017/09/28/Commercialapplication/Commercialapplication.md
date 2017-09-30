---
title: Sharif University Quals CTF 2014 Commercial Application
date: 2017-09-28 16:49:43
tags: 
 - android
---

# Sharif University Quals CTF 2014: Commercial Application

[Github Link](https://github.com/ctfs/write-ups-2014/tree/master/su-ctf-quals-2014/commercial_application/)

> Markdown哪儿都好就是插图太麻烦,尽量用语言描述


## Overview
题目给的是一个Demo小程序,安装在手机或者虚拟机之后看是一个看图程序,一共三张~~黄~~图图,预览版只可以看一张,输入key之后可以看另外两张,提供了一个输入key的窗口,输入错误之后会提示错误(废话),想必正确的key就是题目的flag.

## Analyse
首先例行的对给的apk进行基础操作,我的习惯是同时恢复Java代码和Smali代码.分析的时候Java代码比较方便,出错或者是需要重新打包dex的时候使用Smali.逆向之后发现程序没有lib目录,且Java代码只经过简单混淆,没有报错,说明单纯分析Java代码逻辑即可.

从输入错误key时候给出的报错语句入手,`Your licence key is incorrect...! Please try again with another.`这句在源程序中出现两次,都在`/edu/sharif/ctf/activities/MainActivity.java`之中,查看代码关键位置.

```Java
public void onClick(DialogInterface paramDialogInterface, int paramInt)
      {
        if (KeyVerifier.isValidLicenceKey(this.val$userInput.getText().toString(), MainActivity.this.app.getDataHelper().getConfig().getSecurityKey(), MainActivity.this.app.getDataHelper().getConfig().getSecurityIv()))
        {
          MainActivity.this.app.getDataHelper().updateLicence(2014);
          MainActivity.isRegisterd = true;
          MainActivity.this.showAlertDialog(this.val$context, "Thank you, Your application has full licence. Enjoy it...!");
          return;
        }
        MainActivity.this.showAlertDialog(this.val$context, "Your licence key is incorrect...! Please try again with another.");
      }
```
发现关键逻辑位于KeyVerifier.isValidLicenceKey函数中,对所在文件进行分析,会发现实质性调用的是encrypt函数,该函数需要三个参数.

```Java
public static boolean isValidLicenceKey(String paramString1, String paramString2, String paramString3)
  {
    return encrypt(paramString1, paramString2, paramString3).equals("29a002d9340fc4bd54492f327269f3e051619b889dc8da723e135ce486965d84");
  }
  
public static String encrypt(String paramString1, String paramString2, String paramString3)
  {
    try
    {
      SecretKeySpec localSecretKeySpec = new SecretKeySpec(hexStringToBytes(paramString2), "AES");
      Cipher localCipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
      localCipher.init(1, localSecretKeySpec, new IvParameterSpec(paramString3.getBytes()));
      String str = bytesToHexString(localCipher.doFinal(paramString1.getBytes()));
      return str;
    }
    catch (Exception localException)
    {
      localException.printStackTrace();
    }
    return "";
  }
```

由代码可以发现,加密使用AES算法,对称加密就可逆,现在需要找出使用的三个参数paramString1,paramString2,paramString3,即可逆推出输入的值.回到MainActivity中考察传递三个参数的位置作分析,
`KeyVerifier.isValidLicenceKey(this.val$userInput.getText().toString(), MainActivity.this.app.getDataHelper().getConfig().getSecurityKey(), MainActivity.this.app.getDataHelper().getConfig().getSecurityIv())`
第一个参数是从UI中读入的用户输入,即用户输入的key,第二个是SecurityKey,第三个是SecurityIv,通过调用进一步分析源码包括程序带的SQLite数据库可以找到后两者的值,而AES加密之后的值应该是
`29a002d9340fc4bd54492f327269f3e051619b889dc8da723e135ce486965d84`
这里使用一个~~奇技淫巧~~不去进一步考察代码,而是修改Smali代码,在使用三个值之前将它们从Log中打印出来,条用位置位于`class Ledu/sharif/ctf/activities/MainActivity$4`之中
```Smali
.line 211
    .local v0, "iv":Ljava/lang/String;

    invoke-static {v3, v3}, Landroid/util/Log;->v(Ljava/lang/String;Ljava/lang/String;)I
    invoke-static {v3, v2}, Landroid/util/Log;->v(Ljava/lang/String;Ljava/lang/String;)I
    invoke-static {v3, v0}, Landroid/util/Log;->v(Ljava/lang/String;Ljava/lang/String;)I

    invoke-static {v3, v2, v0}, Ledu/sharif/ctf/security/KeyVerifier;->isValidLicenceKey(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Z

    move-result v1
```
重新打包dex,插入apk中,签名,运行之后随便输入点东西,通过Logcat可以截取三条Log,tag都是随便输入的内容,后两条即SecurityKey,SecurityIv
```
SecurityKey = 37eaae0141f1a3adf8a1dee655853714
SecurityIv = a5efdbd57b84ca36
encrypted = 29a002d9340fc4bd54492f327269f3e051619b889dc8da723e135ce486965d84
```
## Solve
经过以上分析,已经可以基本确定解决问题方法,即利用KeyVerifier中的代码,更改Cipher工作模式为解密,接触答案并ASCII化为字符串,即为key(flag).完整代码如下所示
```Java
package com.company;

import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;

public class Main {

    public static void main(String[] args) {
        // write your code here
        String encrypted = "29a002d9340fc4bd54492f327269f3e051619b889dc8da723e135ce486965d84";
        String securityKey = "37eaae0141f1a3adf8a1dee655853714";
        String securityIv = "a5efdbd57b84ca36";
        String result = decrypt(encrypted, securityKey, securityIv);
        System.out.println(result);
    }

    public static String bytesToHexString(byte[] paramArrayOfByte) {
        StringBuilder localStringBuilder = new StringBuilder();
        int i = paramArrayOfByte.length;
        for (int j = 0; ; j++) {
            if (j >= i)
                return localStringBuilder.toString();
            int k = paramArrayOfByte[j];
            Object[] arrayOfObject = new Object[1];
            arrayOfObject[0] = Integer.valueOf(k & 0xFF);
            localStringBuilder.append(String.format("%02x", arrayOfObject));
        }
    }

    public static String decrypt(String paramString1, String paramString2, String paramString3) {
        try {
            SecretKeySpec localSecretKeySpec = new SecretKeySpec(hexStringToBytes(paramString2), "AES");
            Cipher localCipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
            localCipher.init(Cipher.DECRYPT_MODE, localSecretKeySpec, new IvParameterSpec(paramString3.getBytes()));
            byte[] bytes = localCipher.doFinal(hexStringToBytes(paramString1));
            String flag = "";
            for (byte b : bytes) {
                flag += (char) b;
            }
            return flag;
        } catch (Exception localException) {
            localException.printStackTrace();
        }
        return "";
    }

    public static byte[] hexStringToBytes(String paramString) {
        int i = paramString.length();
        byte[] arrayOfByte = new byte[i / 2];
        for (int j = 0; ; j += 2) {
            if (j >= i)
                return arrayOfByte;
            arrayOfByte[(j / 2)] = (byte) ((Character.digit(paramString.charAt(j), 16) << 4) + Character.digit(paramString.charAt(j + 1), 16));
        }
    }

}

```

运行得到结果  
**Flag:**`fl-ag-IS-se-ri-al-NU-MB-ER`






