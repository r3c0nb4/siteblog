---
layout: post
title: Flush reload on ARM English Version
categories: [Microarchitecture, processor reverse engineering, English version]
description: flush and reload attack on ARM English Version
keywords: Microarchitecture
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Arm A57 flush and reload
The article discusses the application of the flush and reload side-channel attack on ARM chips.
## flush and reload
Flush and reload involves only two simple steps: invalidating the corresponding cache followed by timing. On Intel chips, one can directly locate the flush instruction clflush under user-mode.
```
static inline __attribute__((always_inline)) void clflush(void* p) {
	asm volatile("clflush (%0)\n"::"r"(p));
}
```

Subsequently, one can utilize the rdtsc and rdtscp instructions to calculate the CPU cycles during the process.
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

In ARM architecture, the implementation of flush is not as straightforward. In the paper "Return-Oriented Flush-Reload Side Channels on ARM and Their Implications for Android Devices", although the kernel provides clearcache, it is primarily used to clear the instruction cache (icache) and does not operate on the data cache.

While ARM instruction set does offer some instructions capable of similar operations, the method of achieving cache flush can vary depending on the specific ARM processor architecture. Projects like armageddon (https://github.com/IAIK/armageddon) provide functions for reference, but they may only target certain processors. Implementing cache flush for other processors remains a challenge that requires further investigation and possibly tailored solutions.
```
inline void arm_v8_flush(void* address)
{
  asm volatile ("DC CIVAC, %0" :: "r"(address));
  asm volatile ("DSB ISH");
  asm volatile ("ISB");
}
```

The Linux kernel also provides corresponding system calls for cache flushing operations, as described in this documentation (https://man7.org/linux/man-pages/man2/cacheflush.2.html). However, I am unable to use this header file on my device at the moment, so let's not delve into this discussion for now.

## Arm A8a timing
While ARM does provide instructions to access performance counter registers for obtaining CPU cycle information, armageddon has also implemented relevant functions:
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
However, the MRS instruction consistently displays an "illegal instruction" error on my device, indicating that some devices may not support this instruction. A specific solution to this issue has not yet been found. If a solution is found, I will update this article as soon as possible.
### Alternative
There is a library-provided function that can implement timing functionality, but the smallest unit it supports is nanoseconds. While this may result in some loss of precision, it might still be feasible for the flush and reload technique.
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


