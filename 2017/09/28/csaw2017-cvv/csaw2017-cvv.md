---
title: CSAW 2017 CVV及tabIEZ writeup
date: 2017-09-28 17:25:18
tags:
 - csaw2017
 - misc
 - reverse
---

> 因为需要准备出差, 只做了一些小题



## CVV

信用卡号码算算号器和随机生成器
https://zh.wikipedia.org/zh-hans/Luhn%E7%AE%97%E6%B3%95
从网上找的代码，有问题按照Luhn代码算法说明改


代码很烂，都是粘的还有错。。。
```python
#!/usr/bin/env python

# author: garbu

from random import randint
from pwn import *

def generate_card(type):
    """
    Prefill some values based on the card type
    """
    card_types = ["americanexpress","visa11", "visa16","mastercard","discover"]
    
    def prefill(t):
        # typical number of digits in credit card
        def_length = 16
        
        """
        Prefill with initial numbers and return it including the total number of digits
        remaining to fill
        """
        if 'num:' in t:
            return [int(t[4]), int(t[5]), int(t[6]), int(t[7])], 12
        if t == card_types[0]:
            # american express starts with 3 and is 15 digits long
            # override the def lengths
            return [3, randint(4,7)], 13
            
        elif t == card_types[1] or t == card_types[2]:
            # visa starts with 4
            if t.endswith("16"):
                return [4], def_length - 1
            else:
                return [4], 10
            
        elif t == card_types[3]:
            # master card start with 5 and is 16 digits long
            return [5, randint(1,5)], def_length - 2
            
        elif t == card_types[4]:
            # discover card starts with 6011 and is 16 digits long
            return [6, 0, 1, 1], def_length - 4
            
        else:
            # this section probably not even needed here
            return [], def_length
    
    def finalize(nums):
        """
        Make the current generated list pass the Luhn check by checking and adding
        the last digit appropriately bia calculating the check sum
        """
        check_sum = 0
        
        #is_even = True if (len(nums) + 1 % 2) == 0 else False
        
        """
        Reason for this check offset is to figure out whther the final list is going
        to be even or odd which will affect calculating the check_sum.
        This is mainly also to avoid reversing the list back and forth which is specified
        on the Luhn algorithm.
        """
        check_offset = (len(nums) + 1) % 2
        
        for i, n in enumerate(nums):
            if (i + check_offset) % 2 == 0:
                n_ = n*2
                check_sum += n_ -9 if n_ > 9 else n_
            else:
                check_sum += n
        if check_sum % 10 == 0:
            return nums + [0]
        return nums + [10 - (check_sum % 10) ]
    
    # main body
    t = type.lower()    
    initial, rem = prefill(t)
    so_far = initial + [randint(1,9) for x in xrange(rem - 1)]
    #print "Card type: %s, " % t
    cardnum = "".join(map(str,finalize(so_far))).strip('\n')
    print cardnum
    return cardnum


# # run - check
# generate_card("discover")
# generate_card("mastercard")
# generate_card("americanexpress")

# generate_card("visa13")
# generate_card("visa16")


def ck(nums):
    """
    Make the current generated list pass the Luhn check by checking and adding
    the last digit appropriately bia calculating the check sum
    """
    check_sum = 0
    
    is_even = True if ((len(nums) + 1 )% 2) == 0 else False
    
    """
    Reason for this check offset is to figure out whther the final list is going
    to be even or odd which will affect calculating the check_sum.
    This is mainly also to avoid reversing the list back and forth which is specified
    on the Luhn algorithm.
    """
    check_offset = (len(nums) + 1) % 2
    
    for i, n in enumerate(nums):
        i = int(i)
        n = int(n)
        if i == 0:
            if is_even:
                n_ = n*2
                check_sum += n_ -9 if n_ > 9 else n_
            else:
                check_sum += n
            continue;
        if (i + check_offset) % 2 == 0:
            n_ = n*2
            check_sum += n_ -9 if n_ > 9 else n_
        else:
            check_sum += n
    if check_sum % 10 == 0:
        return 0
    return 10 - (check_sum % 10)


def ck2(nums):
    digits = [int(i) for i in str(nums)]
    odd_digits = digits[-1::-2]
    even_digits = digits[-2::-2]
    total = sum(odd_digits)
    for digit in even_digits:
        total += sum([int(i) for i in str(2 * digit)])
    if total % 10 == 0:
        return "1"
    else:
        return "0"

conn = remote("misc.chal.csaw.io", 8308)
while True:
    cmd = conn.recvline()
    print cmd,

    if 'I need to know if ' in cmd:
        card = "".join(filter(str.isdigit, cmd))
        card = card[:-2]
        print card
        conn.sendline(ck2(card))
        conn.recvline()

    elif 'Visa' in cmd:
        conn.sendline(generate_card('visa16'))
    elif 'MasterCard' in cmd:
        conn.sendline(generate_card('mastercard'))
    elif 'Discover' in cmd:
        conn.sendline(generate_card('discover'))
    elif 'American Express' in cmd:
        conn.sendline(generate_card('americanexpress'))
    elif 'starts with' in cmd:
        num = generate_card("num:" + cmd[-6:-2])
        conn.sendline(num)
    elif 'ends with' in cmd and cmd[-4] in '01234567890':
        num = cmd[-6:-2]
        while True:
            card = generate_card('visa11') + '1' + num
            if ck(card[:-1]) == int(card[-1]):
                print card
                conn.sendline(card)
                break;
    elif 'ends with' in cmd:
        num = cmd[-3:-2]
        while True:
            card = generate_card('visa16')
            if card[-1] == num:
                conn.sendline(card)
                break;

    print conn.recvline(),
   
```
## tabIEZ

Ascii值用表置换，长度不长直接手动查表逐位算出来的

![](tu.jpg)