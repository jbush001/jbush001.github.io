---
layout: post
title: Faster than light
date: '2014-06-06T09:16:00.000-07:00'
author: Jeff
tags:
- performance
- gpgpu
- superscalar
modified_time: '2015-01-04T08:06:10.859-08:00'
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-4679104616393298010
blogger_orig_url: http://latchup.blogspot.com/2014/06/faster-than-light.html
---

In the last post, I discussed the fastest possible execution time of a test
program, the "speed of light," as it were. There was an important assumption
in this calculation: that the CPU issued one instruction per cycle. A common
technique to improve performance in modern processors is issuing multiple
instructions in the same cycle, also known as superscalar. I hadn't put much
thought into a superscalar design given the focus on utilizing the wide vector
unit. However, some helpful comments in a [previous post]({{ site.baseurl }}{%
post_url 2013-12-01-evaluating-instruction-set-tradeoffs %}) have led me to
reconsider this decision.

In this design, there are 16 arithmetic pipelines, one for each vector lane.
For scalar operations, the result from the lowest vector lane is used. When a
scalar operation is issued, the vector unit is not utilized for that cycle. I
instrumented the system and found that for many tests, the vector unit is
underutilized. Ideally, we'd like to keep it as busy as possible with wide
operations. Scalar operations are often just maintenance operations, such as
updating loop counters or advancing pointers.  So, an approach to improving
performance would be to have a separate pipeline to allow co-issuing these
types of simple instructions in parallel with vector instructions.

There are habits that I've developed as a software engineer that sometimes
hinder my efforts designing hardware. I encountered this when thinking about
how to implement this feature. In software, there is a focus on refactoring
and trying to utilize common code. A hallmark of good software design is reuse
of mechanisms. I spent time thinking about how to separate out the scalar
unit, but assumed all scalar operations would use it.

In hardware, redundancy can often improve performance.  One insight is that an
additional pipeline could be made simpler (for example, integer only) and used
_only_ for co-issued instructions. Normally issued scalar instructions would
continue to use the lowest vector pipeline. This would consume a relatively
small amount of area.

However, implementing this and getting it working correctly would require a
fair amount of work.  It would be nice to explore the potential performance
improvement of various design alternatives before implementing it at the RTL
level.

### The Functional Simulator

One of the first things I wrote when I first started working on this GPGPU was
the functional simulator (located in tools/emulator). The simulator, written
in C, is _instruction accurate_ but not _cycle accurate_.  It models
instruction execution properly, but doesn't bother simulating caches, pipeline
hazards, or other implementation details. The functional simulator is a very
useful tool:

  * As a development platform for target software. It is much faster than
    Verilog simulation, which may take minutes to run a program, and much
    easier to set up and debug programs than FPGA. It features a simple
    debugger that can set breakpoints, single step, and inspect memory and
    registers.
  * To run [whole-program compiler tests](https://github.com/jbush001/GPGPU/tree/master/tests/compiler). A
    program (for example, AES encrypt/decrypt) is compiled using the target
    toolchain, run in the simulator, and its output is compared to expected
    patterns.
  * As a golden reference for [hardware verification](https://github.com/jbush001/GPGPU/tree/master/tests/cosimulation). This is sometimes called co-simulation or co-verification. The Verilog
    model is instrumented to dump instruction side effects and the simulator
    compares them to its own execution. Using this mechanism, I can run long
    sequences of randomly generated instructions to root out race conditions
    and hardware bugs. Since the simulator is simpler, it's easier to verify
    its correctness (or, at very least, the bugs in the simulator are likely to
    be different than bugs in the hardware model).
  * For computing rough performance metrics. It's trivial to add performance
    counters to the code or print out when certain events occur.

It is in the last capacity that we will use the simulator for this experiment.

### Results

I added a chunk of code in the simulator to check that two adjacent
instructions could be co-issued. Note that I don't actually simulate co-
issuing them, I just increment a counter when they could be.  Here's the place
where instructions are normally interpreted:

{% highlight c %}
{% raw %}
int retireInstruction(Strand *strand)
{
    unsigned int instr;

    instr = readMemory(strand, strand->currentPc);
    strand->currentPc += 4;
    strand->core->totalInstructionCount++;

+   checkCoIssue(instr, readMemory(strand, strand->currentPc + 4));
{% endraw %}
{% endhighlight %}

I just added some code to check if the instructions are compatible.  The
following rules are currently used:

  * The first instruction must have two vector operands and no mask (because
    masks are stored in scalar registers, we would not have enough read ports
    to issue both at the same time)
  * The second instruction must have two scalar operands

{% highlight c %}
{% raw %}
int coIssued;

void checkCoIssue(unsigned int instr1, unsigned int instr2)
{
    int fmt1 = -1;
    int fmt2 = -1;
    int op2;

    if ((instr1 & 0xe0000000) == 0xc0000000)
        fmt1 = bitField(instr1, 26, 3);
    else if ((instr1 & 0x80000000) == 0)
        fmt1 = bitField(instr1, 28, 3);

    if ((instr2 & 0xe0000000) == 0xc0000000)
    {
        fmt2 = bitField(instr2, 26, 3);
        op2 = bitField(instr2, 20, 6);
    }
    else if ((instr2 & 0x80000000) == 0)
    {
        fmt2 = bitField(instr2, 28, 3);
        op2 = bitField(instr2, 23, 5);
    }

    if (fmt1 == 4 && fmt2 == 0 && op2 <= 0x1f)
        coIssued++;
}
{% endraw %}
{% endhighlight %}

For this test, I used a simple benchmark that roughly simulates bitcoin
hashing (located in benchmarks/hash):

    hash> ../../tools/simulator/simulator WORK/program.hex
    463332 total instructions executed
    394464 vector instructions
    10272 coissued

Without any changes, this program does fairly well with vector unit
utilization: over 85% of instructions are vector instructions. When the coissue
pipeline is added about 2.2% of the instructions are now issued in parallel.
This is not a huge improvement.

My next experiment was to add another scalar read port. This would allow masked
instructions (which this benchmark does not use) and vector instructions with
scalar operands to be co-issued with scalar-only instructions. To simulate
this, I just changed the criteria for fmt1 above. Here are the results:

    hash> ../../tools/simulator/simulator WORK/program.hex
    463332 total instructions executed
    394464 vector instructions
    29444 coissued

That's a little more interesting.  Now over 6% of instructions are co-issued.

### Conclusions

It should be noted that these tests don't consider other pipeline hazards that
would arise from co-issued instructions.  The actual performance improvements
would potentially be smaller than this.  Also, running with only one benchmark
can be misleading. One area I do need to work on is building a larger suite of
performance tests. There are a number of areas to investigate further:

  * Allowing more instruction types to be co-issued. One idea would be to have
    the simulator tabulate joint probabilities for all possible instruction
    pairs in a few test workloads. This would give an indication of which
    pairings would be most beneficial.
  * Modifying [instruction scheduling](http://llvm.org/docs/WritingAnLLVMBackend.html#instruction-scheduling)
    for the the compiler backend to obtain better pairing





