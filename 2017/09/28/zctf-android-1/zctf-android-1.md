---
title: ZCTF Android 1
date: 2017-09-28 16:49:43
tags: 
 - android
---


先做笔记,最后整理成题解.

首先对APK的Java层代码进行分析.入口的逻辑在Login.attemptLogin函数中该函数调用Auth.auth对用户名密码进行校验.

![1](image/1.png)

此处校验有一个读取数据库的函数databaseopt,其逻辑就在下边定义,但是其实不用看,key.db文件就在程序目录中,用随便一个数据库查看器可以看到唯一的键值

![2](image/2.png)

所以,auth函数参数0=context,1=yonghuming+mima,2="zctf2016"
下面分析Auth和auth逻辑.

![3](image/3.png)

Auth有auth,encrypt和decrypt三个函数,后二者是des加解密.auth的逻辑恢复的有些混乱,分析得到黄色部分需要与apk资源中的flag.bin内容进行比较.用python对flag.bin进行des解密还原.盗用程序自带的DES解密,得到结果

```java

public static void main(String[] args) throws Exception {

    String password = "zctf2016";
    File file = new File("C:\\Users\\hyrathon\\Desktop\\Solve\\Android1\\assets\\flag.bin");
    FileInputStream fileInputStream = new FileInputStream(file);
    byte[] str= new byte[fileInputStream.available()];
    fileInputStream.read(str);
    byte[] ans = decrypt(str, password);
    System.out.println(new String(ans));
}
public static byte[] decrypt(byte[] src, String password) throws Exception {
    SecureRandom v3 = new SecureRandom();
    SecretKey v4 = SecretKeyFactory.getInstance("DES").generateSecret(new DESKeySpec(password.getBytes()));
    Cipher v0 = Cipher.getInstance("DES");
    v0.init(2, ((Key)v4), v3);
    return v0.doFinal(src);
}
```
}sihttoN{ftcz,根据原来程序的逻辑,这段是取逆之后的用户名加密码即原来应该为zctf{Notthis},用户名为zctf,密码{Notthis}.到此这部分结![4](image/4.png)束.这一部分目前没有什么影响,因为不计算密码直接改Smali逻辑也可以绕过.检查密码正确后会启动一个新的Activity,密码作为extra参数传给这个Activity,在后面会用到,按下不表.

接下来考察打开的app这个Activity,其中又增加了新的模拟器检验

![5](image/5.png)

task中会结束进程

![6](image/6.png)

修改Smali绕过这处检验
```smali
move-result v0

    if-eqz v0, :cond_c

    .line 106
    const/4 v0, 0x0
    
    invoke-static {v0}, Ljava/lang/System;->exit(I)V
```
if-eqz改为if-nez,绕过此处.

接下来绕过add函数,这个函数是native,检查proc/procid/status中第五项,在android的环境中是tracer

![7](image/7.png)

通过检查该项值判断是否被调试,然而没有什么卵用直接Java层绕过.......话说这个函数伪装成add还传入两个参数,但是并没有用.

接下来pushbottom函数将bottom拷贝到/data/data/com.zctf.app/files/目录下,接下来app调用native函数sayHelloInc,将pass={Notthis}作为参数传入.Java静态部分完成,接下来IDA考察这个函数的功能.

![8](image/8.png)

 上图是关键的一个分支,注意有一个DES解密函数跳转,这里R7储存的是malloc到堆上边的地址,指向解密后的内容,随后两个free,之后就too_late了,所以要在des执行后,free执行前获得r7指向的内存.需要进行动态调试.



![9](image/9.png)

首先考察这里的静态代码,又是熟悉的假add函数,在动态attach上去之后首先修改这里,否则会被发现调试退出,在mov r6,r0下断点强行改变r0=0,接下来执行到free之前,看r7地址,G到内存

![10](image/10.png)

看到了熟悉的PNG头,从这里开始向下把内存内容dump出来.
(这里插两句,ida的android_server只支持arm所以一开始静态分析最好直接从arm开始,该应用lib中包含了x86我没忍住就看了,后来耽误很多时间重新看arm.另外,程序不用的so IDA是检查不到module的,在attach时候加一个"载入新库停下来"然后在其中下断点.)

将复制的内容另存为png图像,使用stegSolve打开图片查图层得到frog(啥?)

![11](image/11.png)


就酱.
http://pan.baidu.com/s/1dEEnQk5
6bjm