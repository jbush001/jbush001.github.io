---
layout: post
title: Messy Details
date: '2014-07-04T09:58:00.000-07:00'
author: Jeff
tags:
- microarchitecture
- performance
- cache coherence
- gpgpu
modified_time: '2015-02-22T07:20:10.838-08:00'
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-7635632076823328356
blogger_orig_url: http://latchup.blogspot.com/2014/07/messy-details.html
---

A problem with books and papers about computer architecture is that they gloss
over messy details. The block diagram looks so simple, but, when you begin to
implement it, you realize there are fundamental structural hazards. They give
no hints about how to handle them. Often, a subtle detail completely alters
the design. I'm going to talk about a lot of details around the cache
hierarchy, specifically the way the L1 and L2 caches communicate with each
other, and a design design with the current microarchitecture I am
implementing.

There are three rules that a "coherent" cache must obey:

  * If a processor writes to a memory location and later reads back from it,
    and no other processors writes to that location in between, it must get the
    same value back.
  * When a processor writes to a location other processors must eventually
    "see" it (read the new value if they access the same address).
  * All processors must see writes in the same order--although not necessarily
    the order written. If one processor reads A then read B from a memory
    location, another processor cannot read B then read A.

There are also rules around memory _consistency_ that give additional
constraints around when written values become visible, but we'll ignore those
for now. Also, this [design](https://github.com/jbush001/NyuziProcessor) has
multiple hardware threads. It is sufficient to replace 'processor' above with
'hardware thread', which, as far as software is concerned, basically looks
like a processor.

## First Generation Microarchitecture

The first version of the microarchitecture has a write-through L1 data cache.
This means memory writes are immediately sent to the L2 cache, which then
updates all of the other cores. The design supports multiple processor cores,
each with their own independent L1 caches, but the cores share an L2 cache.
The write-through design simplifies a lot of things: it adheres to rule 2
automatically because all writes are immediately broadcast.  Rule 3 is
enforced because the L2 can accept only one request per cycle and they are
serialized prior to processing.

But here's the first messy detail. The term 'write through' implies that the
L1 cache line is updated at the same time the response is sent to the L2
cache. That's not actually the case.  If it were, rule #3 would be violated.
This is because threads on the same core share the same L1 data cache. Other
threads on the same core could see the write before threads on other cores,
and the L2 cache may process the transactions in a different order than they
become visible.<sup>1</sup>

Also, having the instruction immediately update the L1 cache would create a
structural hazard: the L2 cache could be broadcasting a write update from
another cache, or filling a L1 read cache miss in the same cycle, which would
result in two things trying to simultaneously update cache memory. This could
be resolved by either adding another write port to the SRAM that contains L1
data--increasing the area--or having one of these transactions fail and retry
--which adds complexity. It would also need to reconcile the case where both
ports simultaneously wrote to the same location. Making the L1 cache only be
updated by the L2 cache resolves this problem very neatly. In fact, this is
how OpenSPARC works.

The design uses a store buffer. When a write occurs, the store value is
latched into a per-thread register. This actually services multiple purposes:

  1. It buffers write requests until they can be sent to the L2 cache. Since
  there are multiple nodes that can access the L2, potentially attempting in
  the same cycle, there are cases where the core must wait to send the request.
  2. It improves performance by allowing the thread to continue running after a
  store is buffered. Subsequent reads check the store buffer and bypass the
  data if the address matches the written address (this enforcing rule #1). If
  another store occurs before the first store is acknowledged, the thread is
  rolled back and suspended.

At first blush, it seems like this design should perform poorly.  Every write
instruction has to go through the L2 interconnect, contending with other
cores, and potentially suspending the thread.  The reason it doesn't is
because writes tend to be relatively infrequent. To demonstrate this, I
compiled chunks of C code from a number of programs (these are actually used
as C compiler tests in the tests/compiler directory) for this architecture and
ran them in the functional simulator. I divided the total number of
instructions executed by the number of store instructions.  The results are
here:

|    | Instructions
|----|----|
| Fannkuch | 7 |
| DCT (libjpeg) | 9 |
| MD5 (ref impl) | 12 |
| adler32 (zlib) | 12 |
| LZSS (zlib) | 14 |
| qsort | 14 |
| AES (ref impl) | 21 |
| crc16 | 28 |

For example, the DCT example issues a store on average every 9 instructions.
Even Fannkuch, which is fairly memory intensive, only issues a store every 7
instructions.  When you consider that the processor does not fully utilize the
pipeline (at best around 80% of cycles issue instructions; the others don't
because of one sort of conflict or another), the actual interconnect
utilization for writes is even lower. When running the teapot renderer from
previous posts in Verilog simulation, I see a store request every 16 cycles on
average.

## Next Generation

When I began implementing the second generation microarchitecture, I
immediately chose a write-back L1 data cache because I felt it would improve
performance and scale to higher core counts.  Admittedly, although I was aware
of the fact that writes are fairly infrequent, I didn't spend much time
collecting data to prove that it would benefit this design.

A write-back cache introduces a lot of extra complexity, because there is now
the potential for caches to have multiple copies of the same memory address in
them. A more complex protocol is required to ensure these stay in sync.  I
chose a snooping MSI protocol with a unidirectional ring bus. A ring bus
requires relatively small area and allows high clock speeds because there are
only point to point links between adjacent nodes.  However, the ring bus adds
its own wrinkles: nodes can see messages in different orders depending on
where they are relative to the sender in the ring. <sup>2</sup>
There is also a delay for messages that are broadcast around the ring, which
creates more potential race conditions.

Despite that, I was fairly pleased with my initial progress.  I got a single
core executing the 3D rendering engine correctly in simulation, and
implemented the cache coherence protocol and state machine. That's when I
started running into implementation details...

**Livelock** In this design, when a cache miss is satisfied, the memory
instruction is not immediately retired. Instead, the thread that caused the
miss is restarted and allowed to retry its operation. There are a few cycles
between the cache line being ready and the thread accessing the cache the
operation. If the cache line is evicted during that time, the operation will
miss the cache again. In the worst case, two cores trying to write the same
cache line could ping pong forever, each invalidating the other then hitting a
cache miss.

The first generation microarchitecture has a similar issue, but the write-back
coherence protocol exacerbates it. In a write-back protocol, there can be only
one cache in the system that has a modified cache line. When a processor wants
to write to an address, it must send a message--a 'write invalidate'--to
remove all other copies. In a write-through architecture, writes always make
progress and read cache lines are only evicted by other read misses (since the
number of hardware threads is the same as the number of cache ways, it's
probably impossible, but I haven't formally proven that).

There are a few ways of addressing the livelock issue in a write-back
architecture:

  1. Apply the write during the same cycle a miss is satisfied so a thread
  always makes progress. On the write side, the store data can be latched in
  the L1 miss queue and applied to the incoming cache line when it is copied
  from the L2 to the L1 cache. On the read side, this gets tricker because the
  L2 data may come in the same cycle another instruction is writing back to the
  register file. OpenSPARC handles this by having two write ports to the
  register file. The second is shared with the multi-cycle hardware divider
  unit. I'd rather not burn an extra register file port just for this edge case.
  2. I also considered having some method of locking the lines into the cache
  until the instruction that caused the cache miss returns. This adds some
  additional complexity, especially around LRU tracking.

**Flow control** Imagine a case--and this is not uncommon--where a processor
begins flushing a large number of modified cache lines its L1 data cache. The
processor can do this much faster than the L2 cache can write back to the
system memory interface (which can required hundreds of cycles). At some point,
the thread doing the flushing needs to be suspended. However, there is no easy
way to do this. The L2 cache could send some kind of NACK message back, but it
would need to track this and be able to resend another message later to cause
the processor to resume execution.

The write-back design also increases area: the L1 tags and data need to be
dual ported to properly support snooping.

## Conclusion

There is value in keeping the design simple.  The write back protocol is a
mess of special cases.  Attempting to verify it's correctness is challenging.
However, part of me doesn't want to abandon it because I've gotten pretty far
along and I'm admittedly kind of enamored with the complexity of it. And,
scalability is a pretty important design aspect for a processor like this.
However, I should probably do a little more modeling and simulation to prove
that the performance gain is really worth it. A lot of manycore designs have
an L2 cache per core (rather than sharing the L2) and implement coherency
between L2 caches.  There may be other bottlenecks that limit scalability in a
shared L2 design. A hybrid design may also be a better alternative: clusters
or four or eight cores could each share an L2 cache and use a write-through
protocol, then the L2 caches could implement a write-back coherence protocol
underneath.

* * *

1. Imagine two cores, A and B, each with two
threads 1 & 2\. Thread A1 writes the value 0xcc to address 0. In the same
cycle, thread B1 writes the value 0x33 to the same location. The L2 cache
sends the updates in order: 0xcc, then 0x33. Thread A2 would see the writes in
order 0xcc, then 0x33. However, thread B2 would see 0x33, 0xcc, 0x33.
2. Consider a ring that has four nodes A -> B
-> C -> D. Messages flow to the right, one station each cycle, with the output
of D looping back to A. Now, in one cycle both A and C simultaneously insert
messages into the ring. Node B will first see the message from A, then the
message from C after it loops around.  However, node D will first see the
message from C, then 2 cycles later, the message from A.

