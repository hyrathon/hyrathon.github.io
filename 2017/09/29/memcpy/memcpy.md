---
title: pwnable.kr之memcpy
date: 2017-09-29 18:00:48
tags: pwnable.kr heap
---

感觉这道题考的是堆分配时字节对齐问题.

首先上代码

```c
// compiled with : gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}

int main(void){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Hey, I have a boring assignment for CS class.. :(\n");
	printf("The assignment is simple.\n");

	printf("-----------------------------------------------------\n");
	printf("- What is the best implementation of memcpy?        -\n");
	printf("- 1. implement your own slow/fast version of memcpy -\n");
	printf("- 2. compare them with various size of data         -\n");
	printf("- 3. conclude your experiment and submit report     -\n");
	printf("-----------------------------------------------------\n");

	printf("This time, just help me out with my experiment and get flag\n");
	printf("No fancy hacking, I promise :D\n");

	unsigned long long t1, t2;
	int e;
	char* src;
	char* dest;
	unsigned int low, high;
	unsigned int size;
	// allocate memory
	char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

	size_t sizes[10];
	int i=0;

	// setup experiment parameters
	for(e=4; e<14; e++){	// 2^13 = 8K
		low = pow(2,e-1);
		high = pow(2,e);
		printf("specify the memcpy amount between %d ~ %d : ", low, high);
		scanf("%d", &size);
		if( size < low || size > high ){
			printf("don't mess with the experiment.\n");
			exit(0);
		}
		sizes[i++] = size;
	}

	sleep(1);
	printf("ok, lets run the experiment with your configuration\n");
	sleep(1);

	// run experiment
	for(i=0; i<10; i++){
		size = sizes[i];
		printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
		dest = malloc( size );

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		slow_memcpy(dest, src, size);		// byte-to-byte memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		fast_memcpy(dest, src, size);		// block-to-block memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
		printf("\n");
	}

	printf("thanks for helping my experiment!\n");
	printf("flag : ----- erased in this source code -----\n");
	return 0;
}
```

程序中自己实现了一个快速的memcpy函数, 题目要求是只要能够跑完所有的测试用例就可以拿到flag, 但是随意输入一些会segment fault.

![](1.PNG)

这个神奇的浮点运算operand movntps没见过, 查一下它的定义, 发现有这样一个报错点:

> The destination operand is a 128-bit or 256-bit memory location. The memory operand must be aligned on a 16-byte (128-bit version) or 32-byte (VEX.256 encoded version) boundary otherwise a general-protection exception (#GP) will be generated.

也就是说它的目的地址需要是0x10对齐的, 但是看我们的程序, 此时操作目标是[edx] = 0x804c4a8, 明显没有0x10对齐, 这就是报错的原因.

回到源码里边溯源, 发现edx这个值是malloc返回值, 也就是说堆上返回的值.

假设我们现在使用这样一个数字序列让程序执行

`8 16 32 64 128 256 512 1024 2048 4096`

通过在malloc函数后面下断点可以查看堆上的布局. 在堆上首先分配0x8的chunk头, 接下来是8位对齐的空间. 由于程序没有调用过free, 所以堆是递增的. 8共分配了8+8=0x10个字节, 它本身是16位对齐的, 所以它的下一个16开始位置是对齐的. 16需要分配8+16=0x18位. 这样一来下一个就不对齐了. 这种效应在64之前没有影响, 因为64位以下调用的是slow_memcpy, 但是在64以上开始调用fast_memcpy, 就会崩溃.

解决这个问题很容易, 在每一个会导致不对齐的数字上+8, 那么就对齐了:

`8 24 40 72 136 264 520 1032 2056 4104`

注意8之后每一个都不对齐, 都加8就好了.



![](2.PNG)

就不直接贴flag了, 做到这里终于把toddler's bottle做完了, 前边的好多题比较简单, 当时做没有写writeup. 喘一口气, 告一段落, 接下来继续前进! 