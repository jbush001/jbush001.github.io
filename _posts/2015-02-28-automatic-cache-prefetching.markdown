---
layout: post
title: Automatic Cache Prefetching
date: '2015-02-28T19:58:00.000-08:00'
author: Jeff
tags:
- automatic cache prefetch
- microarchitecture
- performance
- profiling
modified_time: '2015-06-18T18:31:54.645-07:00'
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-7919053102990799020
blogger_orig_url: http://latchup.blogspot.com/2015/02/automatic-cache-prefetching.html
---

Many optimizations take the form of "lazy evaluation."  When an operation may
end up not being needed, we defer it as long as possible for the chance of
avoiding it entirely. However, there is another class of optimizations which
attempt to do work speculatively before we know that we need to. At first
blush, it seems like there are a few conditions where it might be advantageous
to do this:

1. If the operation has a lot of latency.
2. If the resource that is needed for the operation tends to be
underutilized.
3. The operation ends up being needed most of the time anyway.

I've experimented with automatically prefetching cache lines to improve
performance in my [GPGPU](https://github.com/jbush001/NyuziProcessor) design.
This is a common technique in many computer architectures. When a cache miss
occurs, it will automatically read a few of following cache lines.

### Implementation

I've implemented an the prefetch mechanism in the L2 cache for simplicity.
This means that accesses to prefetched lines may still cause an L1 miss.
However, since L1 misses have relatively low latency compared to external
memory accesses, this shouldn't matter much. I believe the following analysis
supports this conclusion.

The first stage in the [L2 cache pipeline](https://github.com/jbush001/NyuziPr
ocessor/wiki/Microarchitecture#l2-cache) is a convenient place to put the
prefetch mechanism: it can snoop cache misses as they flow by, and can inject
memory requests when no other requests are pending.

The two bolded chunks of SystemVerilog code below represent the prefetch logic
that has been added to l2_cache_arb. The first chunk detects when prefetching
should be started after a cache miss.  If another cache miss occurs while a
prefetch is already active (which is not uncommon), it will be ignored.

The second chunk injects prefetch requests into the pipeline. This clause is
only active if there aren't requests from cores or restarted fills. Prefetches
are opportunistic, so this ensures they don't starve normal requests.

<pre>
    if (l2bi_ready)
    begin
        // Restarted request from external bus interface
        ...
<b>        if (!prefetch_active && l2bi_request.packet_type != L2REQ_PREFETCH)
        begin
            prefetch_active <= 1;
            prefetch_address <= l2bi_request.address + `CACHE_LINE_BYTES;
        end
</b>
    end
    else if (|grant_oh && can_accept_request)
    begin
        // New request from a core
        ...
    end
<b>    else if (prefetch_active && can_accept_request)
    begin
        // Inject prefetch commands
        prefetch_address <= prefetch_address + `CACHE_LINE_BYTES;
        prefetch_count <= prefetch_count + 1;
        if (prefetch_count == PREFETCH_COUNT - 1)
            prefetch_active <= 0;

        l2a_request.valid <= 1;
        l2a_request.packet_type <= L2REQ_PREFETCH;
        l2a_request.address <= prefetch_address;
        l2a_is_l2_fill <= 0;
    end
</b>
</pre>

### Validation


Being an optimization, this is the sort of feature that can appear to work
correctly, but not actually be doing anything. After running regression tests
to make sure this change didn't break normal functionality, I attempted to
prove that this was indeed prefetching as intended.  My first step was to
instrument the code with some $display statements.  In the output, I can see
the prefetches being injected and completing:

    filled cache miss 00200000
    inserting prefetch 00200040
    inserting prefetch 00200080
    inserting prefetch 002000c0
    inserting prefetch 00200100
    filled prefetch 00200040
    filled prefetch 00200080
    filled prefetch 002000c0
    filled prefetch 00200100
    ...

This looks correct.  Next, I ran a single threaded memory read test.  Here are
the results:

<table border="1px">
<tbody>
<tr><th></th><th>No Prefetch</th><th>Prefetch
</th></tr>
<tr><th>Total Cycles</th><td>846,091</td><td>574,293</td></tr>
<tr><th>l2 miss</th><td>16,390</td><td>4,688</td></tr>
<tr><th>l2 hit</th><td>3</td><td>14,317</td></tr>
</tbody></table>

This looks correct.  Prefetching makes this program run 32% faster (looking at
total cycles). The L2 cache miss rate has decreased from 99.98% to 24%.

### Things Get More Complicated

This design is intended to run programs that are heavily multithreaded, so the
case above is not representative of typical workloads. The next step is to run
the multithreaded read benchmark. This test uses four hardware threads to read
interleaved chunks of memory, and uses vector loads to read an entire cache
line at a time.

<table border="1px"><tbody>
<tr><th></th><th>No Prefetch</th><th>Prefetch</th></tr>
<tr><th>Total Cycles</th><td>459,914</td><td>467,098</td></tr>
<tr><th>l2 miss</th><td>16,394</td><td>9,005</td></tr>
<tr><th>l2 hit</th><td>5</td><td>8,996</td></tr>
</tbody></table>

The L2 miss rate has decreased with prefetch enabled, which is what we would
expect.  However, this is a dead heat, with the prefetch being about 1.5%
slower.  Based on previous posts, that shouldn't be a surprise.  Hardware
multithreading is already hiding latency, so the prefetch mechanism is
redundant.

The next case I tried is the write benchmark.  It uses vector wide writes and
multiple threads as above.

<table border="1px"><tbody>
<tr><th></th><th>No Prefetch</th><th>Prefetch</th></tr>
<tr><th>Total Cycles</th><td>462,041</td><td>721,704</td></tr>
<tr><th>l2 miss</th><td>16,380</td><td>9,222</td></tr>
<tr><th>l2 hit</th><td>2</td><td>8,488</td></tr>
</tbody></table>

Although the L2 cache miss rate has decreased here as well, the performance is
around 55% _slower_. This occurs because of a bad interaction with a hardware
optimization.

When a cache miss occurs writing a 32-bit word to memory, the hardware first
loads the entire cache line (64 bytes) into the L2 cache. It then updates the
32-bit word that was written in that cache line and marks the line dirty.  It
will be written back to memory when that line needs to be evicted to make room
for other lines. However, this architecture allows an entire vector to be
stored at an aligned address with a single instruction. Vector registers are
the same size as a cache line (this is not a coincidence).  In the case where
a vector write misses the cache, loading the line from memory first would be a
waste, since it's going to be entirely overwritten anyway. There is logic to
detect this condition and skip the memory load. The write benchmark is using
vector writes.

However, automatic cache prefetching sabotages this optimization by
prefetching those lines anyway. This is demonstrated by adding a little
Verilog code to count the number of external memory references:

<table border="1px"><tbody>
<tr><th></th><th>No Prefetch</th><th>Prefetch</th></tr>
<tr><th>External Memory Reads</th><td>5</td><td>8453</td></tr>
</tbody></table>

### A More Realistic Workload

The tests so far show that the performance of this feature varies wildly.  In
the best case, it improved performance by 32%, but in the worst, it degraded
it by 55%.  It also seems to be worse with more highly optimized code.
However, these are all very specific microbenchmarks.  Let's grab some numbers
from a more mixed test case, the 3D rendered textured cube from previous
posts:

<table border="1px"><tbody>
<tr><th></th><th>No Prefetch</th><th>Prefetch</th></tr>
<tr><th>Total Cycles</th><td>17,808,761</td><td>20,742,364</td></tr>
<tr><th>l2 miss</th><td>102,058</td><td>63,684</td></tr>
<tr><th>l2 hit</th><td>307,285</td><td>348,439</td></tr>
</tbody></table>

As before, prefetch reduces the l2 miss rate, but the performance hit is
pretty significant: 16% slower with prefetch enabled.

### Conclusions

Overall, this feature seems to be detrimental to performance. Why? I initially
speculated that there are three rules that define when speculative work is
beneficial.  Rule 1 states there needs to be high latency. However, one could
argue that hardware threading has already hidden this latency and thus it is
no longer true.  We didn't really measure memory interface utilization in this
test, although it's probably safe to infer that hardware multithreading also
improves memory utilization. Rule 3 states that the operation should occur
most of the time anyway. In the most degenerate benchmarks I ran, this is not
true: the vector write optimization skips the operation often in some cases.

There are other experiments I could have run.  For example, it might be
interesting to disable the vector write optimization to see if there was a
difference between prefetch and no-prefetch modes--which would support the
theory that the interaction with that optimization is primarily penalizing the
mixed use case. I also could have changed the number of prefetched cache lines
to see if that made a difference.  However, the basic numbers seemed bad
enough that it's probably not worth it.
