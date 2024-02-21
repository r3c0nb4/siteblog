---
layout: post
title: Cpu cache English version
categories: [Microarchitecture, processor reverse engineering, English version]
description: Cache
keywords: Microarchitecture
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# CPU cache
The cache sits between the main memory (DRAM) and the processor registers, serving to reduce the latency of memory load and store operations. The structure of a cache is roughly as follows:

![avatar](/images/posts/cache-arch.png)

## Details about cache
The minimum unit of cache is the cache line, also referred to as a block. Every time the processor fetches data or instructions from memory into the cache, it does so in blocks of a certain size, known as the block size. Generally, there are no accesses smaller than the block size.

In Linux, you can use the following command to view the block size:

```
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

In modern computer architectures, memory is typically mapped to the cache using a set-associative approach. A set consists of a group of cache lines, each containing several cache blocks. In a design with 4-way set associativity, for example, a set would contain four cache lines.

All blocks in memory are divided into different cache groups. When a memory address is accessed, it is first hashed or indexed to determine which set it belongs to. Then, within that set, a specific cache line is selected for storing the data. This set-associative mapping helps to improve cache performance by reducing conflicts and allowing for more efficient use of cache space.

![avatar](/images/posts/cache-set-associative.png)

In a group, each memory block mapped can place its data into any cache line within that group. For example, in a 4-way set-associative design, each memory block can be mapped in four different ways, meaning it can be stored in any one of the four cache lines within that group. This flexibility allows the cache to selectively utilize different cache lines when storing data, reducing cache line contention and improving cache utilization efficiency.

## tags
Physical memory addresses roughly consist of three parts: the tag, set, and offset. The offset is used to address different bytes within a block, while the set corresponds to the mapped cache set. The tag is used to differentiate which memory block each cache line within the same cache set belongs to.

For example, suppose a physical memory set is 1 and the tag is 2. To determine if a block is in the cache, the processor performs the following operations: it uses the set to locate the corresponding cache group, then searches within the cache group to see if there is a cache line with a tag equal to 2. If it finds a tag equal to 2, and the cache line is valid, it's considered a cache hit, and the processor retrieves the data directly from the cache. If it doesn't find this tag, the processor needs to fetch the data from memory.

## Prefetcher
Cache miss remains a significant factor affecting processor performance. Frequent cache misses require the processor to frequently access memory to fetch data, significantly slowing down the speed of execution. To address this issue, processors incorporate a module called a prefetcher, which works to proactively fetch data into the cache before it's actually needed.

The prefetcher monitors memory access patterns and predicts which data will be needed in the near future based on these patterns. It then fetches this data from main memory into the cache before the processor actually requests it. By doing so, the prefetcher helps to reduce the likelihood of cache misses and minimize the performance impact of memory access latencies.

### Software prefetcher
Some compilers may include prefetcher instructions, for example:

```
// without prefetcher
for (int i=0; i<1024; i++) {
    array1[i] = 2 * array1[i];
}

// with prefetcher
for (int i=0; i<1024; i++) {
    prefetch (array1 [i + k]);
    array1[i] = 2 * array1[i];
}
```
### Stream prefetcher
This type of prefetcher may be more widely applied. Specifically, the prefetcher records the direction of memory accesses (possibly within a single page, although not entirely certain at this point, but not very critical). For example, if `c = buffer[X]` causes a cache miss, this cache miss will be recorded by the prefetcher window. Subsequently, if c = buffer[X + K] results in another cache miss, and finally another cache miss occurs with `c = buffer[X + K + M]`, the prefetcher will roughly determine the memory access direction to be additive. Consequently, data larger than this address will be fetched into the cache. The size of the fetched data is uncertain and may vary depending on different processor designs.

One can exploit the flush-reload side-channel attack to observe the behavior of the stream prefetcher:
```
  pick = reloadbuffer[0];
		for(int i = 0 ; i < probe_size; i++){
			clock_init(&start);
			pick = reloadbuffer[i];
			clock_end(&end);
			time = get_clock(start, end);
    }
```

I accessed a memory region of size equal to a page size and outputted the occurrence of each cache miss.
```
r3c0n@ubuntu:~/project$ cat output 
i: 64 addr: 0x0x2000001000, time: 192
i: 128 addr: 0x0x2000002000, time: 192
i: 192 addr: 0x0x2000003000, time: 128
i: 256 addr: 0x0x2000004000, time: 160
i: 320 addr: 0x0x2000005000, time: 160
i: 384 addr: 0x0x2000006000, time: 160
i: 448 addr: 0x0x2000007000, time: 160
```
Although I initially accessed `reloadbuffer[0]`, the next block was not fetched into the cache. After seven cache misses, the prefetcher started working and fetched the remaining data into the cache.

