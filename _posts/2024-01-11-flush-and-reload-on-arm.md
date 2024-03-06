---
layout: post
title: Flush reload on ARM
categories: [Microarchitecture, processor reverse engineering]
description: flush and reload attack on ARM
keywords: Microarchitecture
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Arm A57 flush and reload
本文讨论 flush and reload 测信道在 Arm 芯片上的应用。
## flush and reload
Flush 和 reload 只有两个简单步骤： 将对应的 cache invalidate，随后进行计时。 在 Intel 芯片中，可以直接找到 user-mode 下的 flush 指令 clflush。

```
static inline __attribute__((always_inline)) void clflush(void* p) {
	asm volatile("clflush (%0)\n"::"r"(p));
}
```

随后就可以用 rdtsc 指令和 rdtscp 指令计算期间的 cpu cycle。
```
static inline __attribute__((always_inline)) uint64_t rdtsc(void) {
	uint64_t lo, hi;
	asm volatile("rdtsc\n" : "=a" (lo), "=d" (hi) :: "rcx");
	return (hi << 32) | lo;
}

static inline __attribute__((always_inline)) uint64_t rdtscp(void) {
	uint64_t lo, hi;
	asm volatile("rdtscp\n" : "=a" (lo), "=d" (hi) :: "rcx");
	return (hi << 32) | lo;
}

static inline __attribute__((always_inline)) void cpuid(void) {
	asm volatile ("cpuid\n\t" ::: "%rax", "%rbx", "%rcx", "%rdx");
}
```

## Arm A8a flush
在 Arm 寄存器上，flush的实现没有这么简单。在论文 Return-Oriented Flush-Reload Side Channels on ARM and Their Implications for Android Devices
中，尽管内核实现了 `clearcache` ，但是仅仅用来清理 icache 中的指令缓存，对于 data cache 不做操作。

但是 Arm 指令集也存在一些指令可以做同样的操作。 https://github.com/IAIK/armageddon 提供了相关的函数可以参考，但是仅仅针对一部分处理器，其他处理器如何实现 cache flush 仍然是个问题。
```
inline void arm_v8_flush(void* address)
{
  asm volatile ("DC CIVAC, %0" :: "r"(address));
  asm volatile ("DSB ISH");
  asm volatile ("ISB");
}
```

https://man7.org/linux/man-pages/man2/cacheflush.2.html，Linux 内核也提供了相应的系统调用可以 flush cache 。但是在我的设备上无法使用这个头文件，暂时不做讨论。

## Arm A8a timing
Arm 虽然提供了对应的指令可以访问 performance counter 寄存器从而获取 cpu cycle， armageddon也实现了相关函数：
```
inline void
arm_v8_timing_init(void)
{
  uint32_t value = 0;

  /* Enable Performance Counter */
  asm volatile("MRS %0, PMCR_EL0" : "=r" (value));
  value |= ARMV8_PMCR_E; /* Enable */
  value |= ARMV8_PMCR_C; /* Cycle counter reset */
  value |= ARMV8_PMCR_P; /* Reset all counters */
  asm volatile("MSR PMCR_EL0, %0" : : "r" (value));

  /* Enable cycle counter register */
  asm volatile("MRS %0, PMCNTENSET_EL0" : "=r" (value));
  value |= ARMV8_PMCNTENSET_EL0_EN;
  asm volatile("MSR PMCNTENSET_EL0, %0" : : "r" (value));
}

inline void
arm_v8_timing_terminate(void)
{
  uint32_t value = 0;
  uint32_t mask = 0;

  /* Disable Performance Counter */
  asm volatile("MRS %0, PMCR_EL0" : "=r" (value));
  mask = 0;
  mask |= ARMV8_PMCR_E; /* Enable */
  mask |= ARMV8_PMCR_C; /* Cycle counter reset */
  mask |= ARMV8_PMCR_P; /* Reset all counters */
  asm volatile("MSR PMCR_EL0, %0" : : "r" (value & ~mask));

  /* Disable cycle counter register */
  asm volatile("MRS %0, PMCNTENSET_EL0" : "=r" (value));
  mask = 0;
  mask |= ARMV8_PMCNTENSET_EL0_EN;
  asm volatile("MSR PMCNTENSET_EL0, %0" : : "r" (value & ~mask));
}

inline void
arm_v8_reset_timing(void)
{
  uint32_t value = 0;
  asm volatile("MRS %0, PMCR_EL0" : "=r" (value));
  value |= ARMV8_PMCR_C; /* Cycle counter reset */
  asm volatile("MSR PMCR_EL0, %0" : : "r" (value));
}
```
但是 `MRS` 指令在我的设备上一直显示 illegal instruction ， 可能一些设备不支持这个指令。具体的解决方法暂时还没有找到，要是能找到会尽快更新这篇文章。

破案了, 现在更新一下这一段, illegal instruction 不是因为设备不支持,我们是可以通过其他方法使用performance counters的.
其实也很简单, 我们需要一个内核模块enable performance counter
```
#include <linux/kernel.h>
#include <linux/module.h>

static void enable(void *in)
{
	u64 cycles;

	asm volatile("msr pmuserenr_el0, %0" : : "r"(BIT(0) | BIT(2)));
	asm volatile("mrs %0, pmcr_el0" : "=r" (cycles));
	cycles |= (BIT(0) | BIT(2));
	asm volatile("msr pmcr_el0, %0" : : "r" (cycles));
	cycles = BIT(27);
	asm volatile("msr pmccfiltr_el0, %0" : : "r" (cycles));
}

static void disable(void *in)
{
	asm volatile("msr pmcntenset_el0, %0" :: "r" (0 << 31));
	asm volatile("msr pmuserenr_el0, %0" : : "r"((u64)0));

}

static int __init init(void)
{
	on_each_cpu(enable, NULL, 1);
	return 0;

}

static void __exit fini(void)
{
	on_each_cpu(disable, NULL, 1);
}

module_init(init);
module_exit(fini);
```

随后就可以使用performance counter了
```
static inline uint64_t cycles(void)
{
	uint64_t val;
	asm volatile("mrs %0, pmccntr_el0" : "=r"(val));
	return val;
}

static uint64_t cycles_read(volatile uint8_t *addr){
	volatile uint8_t pick = 0;
	uint64_t cycles1 = 0, cycles2 = 0;
	
	cycles1 = cycles();	
	pick = *addr;
	cycles2 = cycles();
	barrier();
	return cycles2 - cycles1;
}
```

### Alternative
有一个库提供的函数可以实现记时功能，但是最小的单位是 nanosecond，尽管精度上会差一些，但是对于 flush and reload 也许可行。

https://man7.org/linux/man-pages/man3/clock_gettime.3.html

```
#include <time.h>
#include <stdint.h>
#define BILLION 1000000000L

void clock_init(struct timespec *start){
  clock_gettime(CLOCK_REALTIME, start);
}

void clock_end(struct timespec *end){
  clock_gettime(CLOCK_REALTIME, end);
}

uint64_t get_clock(struct timespec start, struct timespec end){
  return BILLION * (end.tv_sec - start.tv_sec) + end.tv_nsec - start.tv_nsec;
}
```


