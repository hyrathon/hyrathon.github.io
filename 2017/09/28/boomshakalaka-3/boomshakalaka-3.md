---
title: 0CTF 2016 boomshakalaka-3
date: 2017-09-28 16:49:43
tags: 
 - android
---

# 0CTF : boomshakalaka-3

**Category:** Mobile
**Points:** 3
**Solves:** 35
**Description:**

> play the game, get the highest score
>


## Write-up

脑洞题, 基础的分析之后可以发现是由Cocos2d写的游戏程序, 部分与flag有关的代码在Java层, 在sharedPreference里边写了MGNO, 进一步的写sharedPreference在native代码中寻找.

从updateScore函数中可以发现根据分数写flag的明显痕迹, 其他方法中也有一部分写的代码, 但是顺序未知.

```C
    if ( v3 == (const char *)100 )
    {
      v6 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v22, &v20, "MW");
      cocos2d::CCUserDefault::setStringForKey(v6, &v34, &v22);
      v7 = &v22;
    }
    else if ( v3 == (const char *)600 )
    {
      v8 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v23, &v20, "Rf");
      cocos2d::CCUserDefault::setStringForKey(v8, &v34, &v23);
      v7 = &v23;
    }
    else if ( v3 == (const char *)700 )
    {
      v9 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v24, &v20, "Rz");
      cocos2d::CCUserDefault::setStringForKey(v9, &v34, &v24);
      v7 = &v24;
    }
    else if ( v3 == (const char *)3000 )
    {
      v10 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v25, &v20, "Bt");
      cocos2d::CCUserDefault::setStringForKey(v10, &v34, &v25);
      v7 = &v25;
    }
    else if ( v3 == (const char *)5600 )
    {
      v11 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v26, &v20, "RV");
      cocos2d::CCUserDefault::setStringForKey(v11, &v34, &v26);
      v7 = &v26;
    }
    else if ( v3 == (const char *)9900 )
    {
      v12 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v27, &v20, "9Z");
      cocos2d::CCUserDefault::setStringForKey(v12, &v34, &v27);
      v7 = &v27;
    }
    else if ( v3 == (const char *)18000 )
    {
      v13 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v28, &v20, "b1");
      cocos2d::CCUserDefault::setStringForKey(v13, &v34, &v28);
      v7 = &v28;
    }
    else if ( v3 == (const char *)88800 )
    {
      v14 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v29, &v20, "Vf");
      cocos2d::CCUserDefault::setStringForKey(v14, &v34, &v29);
      v7 = &v29;
    }
    else if ( v3 == (const char *)100000 )
    {
      v15 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v30, &v20, "S2");
      cocos2d::CCUserDefault::setStringForKey(v15, &v34, &v30);
      v7 = &v30;
    }
    else
    {
      if ( v3 != (const char *)1000000000 )
      {
LABEL_25:
        v17 = cocos2d::CCString::createWithFormat((cocos2d::CCString *)"%d", v3);
        (*(void (__fastcall **)(_DWORD, _DWORD))(**(_DWORD **)(v18 + 264) + 428))(
          *(_DWORD *)(v18 + 264),
          *(_DWORD *)(v17 + 20));
        return sub_3A1DDC(&v20);
      }
      v16 = cocos2d::CCUserDefault::sharedUserDefault(v5);
      std::operator+<char,std::char_traits<char>,std::allocator<char>>(&v31, &v20, "4w");
```

打上一局游戏, root权限查看sharedpreference, 可以发现部分flag:MGN0ZntDMGNvUzJkX0FuRHJv**MWRf**dz99

注意加粗的地方是与分数先关的, 其余的或许与前面或许与后面有关, 我们不敢确定, 但是我们可以把缺少的分数部分补全.MGN0ZntDMGNvUzJkX0FuRHJvMWRfRzBtRV9Zb1VfS24wdz99, 解码得到0ctf{C0coS2d_AnDro1d_G0mE_YoU_Kn0w?}.

## Other write-ups and resources

* <https://eugenekolo.com/blog/0ctf-2016-boomshakalaka-writeup/>
* [P4 Team](https://github.com/p4-team/ctf/blob/master/2016-03-12-0ctf/boomshakalaka/README.md)
