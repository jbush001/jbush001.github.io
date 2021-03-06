---
layout: post
title: Further Reading
date: '2016-09-04T08:39:00.000-07:00'
author: Jeff
tags:
- hardware
- 3d rendering
- gpu
modified_time: '2016-09-13T06:47:43.318-07:00'
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-8350511321799951226
blogger_orig_url: http://latchup.blogspot.com/2016/09/further-reading.html
---

A passage from Mortimer J. Adler and Charles Van Doren's "[How to Read a
Book](https://www.goodreads.com/book/show/567610.How_to_Read_a_Book),"
discusses reading for understanding:

> "If the book is completely intelligible to you from start to finish, then
the author and you are as two minds in the same mold. The symbols on the page
merely express common understanding you had before you met.
>
> Let us take our second alternative. You do not understand the book
perfectly. Let us even assume--what unhappily is not always true--that you
understand enough to know that you do not understand it all. You know the book
has more to say than you understand and hence it contains something that can
increase your understanding...
>
> Without external help of any sort, you go to work on the book. With nothing
but the power of your own mind, you operate on the symbols before you in such
a way that you gradually lift yourself from a state of understanding less to
one of understanding more. Such elevation, accomplished by the mind working on
a book, is highly skilled reading, the kind of reading that a book which
challenges your understanding deserves"

I think you could replace the word "book" with "source code" above and it
would be equally applicable. However, although I've spent a lot of time
reading source code for other people's programs, most of the time it doesn't
feel like a highly skilled activity. It is often confusing and frustrating.
That said, it is a necessary and useful way to learn, perhaps as much as
writing code.

Unfortunately for those interested in hardware architecture, and especially
GPUs, source code is hard to come by and what exists of written documentation
is often unhelpful. Although GPU manufacturers publish academic papers in
journals and at conferences about their new architectures, they are often
glorified marketing white papers with irritatingly little detail. Many PC
review sites also present architectural "deep dives," but they're like reading
Car and Driver when what you really want is a [Haynes
Manual](https://haynes.com/en-us/). I'm not interested in memorizing the
displacement or horsepower of various models: I want to know how the tappets
are assembled.

Recently, I posted a high level analysis of a few commercial GPU
implementations: the [VideoCore IV GPU]({{ site.baseurl }}{% post_url
2016-02-26-life-of-triangle %}) and [GPLGPU]({{ site.baseurl }}{% post_url
2016-07-24-gplgpu-walkthrough %}). I've found a few other interesting GPU
focused projects and documentation that, while requiring some investment to
understand, are worth serious study. Many of these include implementations
that anyone can get running on their own machine in simulation. Being able to
add print statements to a program see the sequence of operations and modify it
to see how its behavior changes (or breaks) is a useful way to understand how
it works.

### A Trip Through the Graphics Pipeline

[https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index/](https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-
pipeline-2011-index/)

Fabian Giesen's well-written blog is full of interesting posts about computer
hardware, programming, and graphics, but his 13 part series about the GPU
pipeline is essential reading if you're interested in how GPUs work. It walks
through the pipeline for a desktop class GPU with explanations of algorithms
at each stage.

Explaining something clearly necessarily requires simplification, and it is a
tough balance to keep things clear while not oversimplifying to the point of
creating more confusion. I think this series does a great job, elaborating
where interesting and useful, and calling out where it has omitted details. It
forms a good foundation for further study.

### MilkyMist One

[https://github.com/m-labs/milkymist](https://github.com/m-labs/milkymist)

MilkyMist was a commercial VJ console released in 2010, used for displaying
trippy visualizations synchronized to music on large projection screens at
parties. It's visual effects are based on the MilkDrop WinAmp visualizer
plugin. It contains a system-on-chip running on FPGA, including a CPU (based
on Lattice Micro's open source LM32, with modifications to support virtual
memory), a high performance memory subsystem, and peripherals for A/V output
and input. The source code is modular and well organized. The components can
be run on FPGA or in Verilog simulation.

What's most impressive about this project is that it operates at full frame
rates using modest FPGA hardware. Examining optimized implementations is
useful, because many of the things you need to do to get good performance have
deep design impacts.

While it may not look like a GPU at first blush, it has a graphics pipeline to
support visual effects that is similar to a fixed function 3D pipeline.

[https://github.com/m-labs/milkymist/tree/master/cores/tmu2](https://github.com/m-labs/milkymist/tree/master/cores/tmu2)

In the top level module tmu2.v, there are many blocks that look familiar from
analysis of VideoCore and GPLGPU:

  * **Vertex fetch** (tmu2_fetchvertex): reads vertex information from system
    memory and sends to the next pipeline stage. Milkymist performs image
    warping by breaking the source image into a grid of quadralaterals and
    adjusting the output coordinates. The VideoCore IV GPU also has a vertex
    engine that reads vertex attributes.
  * **Parameter interpolation.**The modules
    tmu2_vdivops/tmu2_vdiv/tmu2_hdivops/tmu2_hdiv appear to perform setup for
    the texture interpolators, and the modules tmu2_vinterp/tmu2_hinterp
    perform interpolation to generate the source texture coordinates. The
    VideoCore IV had a hardware interpolator block, as did GPLGPU. MilkyMist is
    slightly simpler because it doesn't need to worry about perspective
    correction.
  * **Texture Sampling** (tmu2_adrgen/tmu2_texmem/tmu2_blend) This is based on
    another influential paper I had spent some time reading: [Prefetching in a
    Texture Cache
    Architecture](https://graphics.stanford.edu/papers/texture_prefetch/texture_
    prefetch_comp.pdf). It's helpful to see a clean implementation of this
    algorithm. It would be interesting to compare this in more detail with the
    texture cache in GPLGPU, which was implemented before that paper came out.
  * **Alpha Blending/Writeback** The modules
    tmu2_fdest/tmu2_alpha/tmu2_burst/tmu2_pixout handle alpha blending and
    memory access. This operates similar to the memory controller I discussed
    in the [GPLGPU walkthrough]({{ site.baseurl }}{% post_url
    2016-07-24-gplgpu-walkthrough %}). Alpha blending is a read/modify/write
    operation, so tmu2_fdest fetches the destination pixels, tmu2_alpha
    combines the old pixels with new, tmu2_burst collects a burst worth of
    pixels and tmu2_pixout writes them back to memory.

There are other modules specific to the MilkDrop effect like tmu2_decay.

The author's [master's thesis](https://m-labs.hk/thesis/thesis.pdf) on this
project is well worth reading. It discusses the challenges of getting high
throughput from the memory subsystem and details the implementation of many
components.

### Attila

[http://attila.ac.upc.edu/wiki/index.php/Main_Page](http://attila.ac.upc.edu/wiki/index.php/Main_Page)\\
[http://vmoya.site.ac.upc.edu/docs/ISPASS%20-%20ATTILASim.pdf](http://vmoya.site.ac.upc.edu/docs/ISPASS%20-%20ATTILASim.pdf)\\
[https://github.com/attila-gpu/attila-sim](https://github.com/attila-gpu/attila-sim)

A group of PhD students at the Polytechnic University of Catalonia started the
Atilla project to develop a microarchitecture for a desktop class GPU. It has
been dormant since 2010, but is quite functional in its current form. It has a
cycle-accurate C simulator of a full GPU pipeline, including both programmable
and fixed function units. It does not have a synthesizable hardware
implementation. It supports programmable shaders using the older [ARB shader
syntax](https://en.wikipedia.org/wiki/ARB_assembly_language). The project also
has a shim library that can capture OpenGL/DirectX 9 commands from games
running on a desktop machine and replay them through their simulator. The wiki
has impressive (for their era) frame dumps rendered from games like Crysis,
Half Life 2, and Call of Duty 2, which demonstrates that they have a fully
functional system.

This architecture allows both non-unified and unified shader models (in the
former, there are dedicated shaders for vertex and fragment shader. The latter
uses the same pool for both). Like GPLGPU, it is an immediate mode renderer:
it writes completed pixels directly to dedicated graphics memory rather than
an intermediate tile buffer.

*(a note in the Wiki indicates that it unfortunately may disappear soon
because one of the authors is no longer a student at the university that is
hosting it).*

### GPGPU-SIM

[http://www.gpgpu-sim.org/](http://www.gpgpu-sim.org/)\\
[https://github.com/gpgpu-sim/](https://github.com/gpgpu-sim/)\\
[http://www.academia.edu/download/3239006/gpgpusim.ispass09.pdf](http://www.academia.edu/download/3239006/gpgpusim.ispass09.pdf)

This is a C simulator for the compute unit for a modern GPU implemented by a
research group at the University of British Columbia. It is influenced by
NVidia's architecture and can run CUDA programs written in [PTX intermediate
code.](http://docs.nvidia.com/cuda/parallel-thread-execution/) It has detailed
documentation and includes a preconfigured virtual machine image to simplify
setup. It does not include any fixed function graphics units, so it can't
simulate graphics rendering out of the box. It is not a synthesizable hardware
implementation, but is cycle accurate and includes a detailed energy model.

A lot of GPU block diagrams and descriptions seem to imply that shaders are
just simple CPUs. This project highlights how untrue that is, with specialized
hardware to handle a number of functions:

  * Branch divergence. GPGPU-sim uses a stack of execution masks, described
    [here](http://gpgpu-sim.org/manual/index.php/Main_Page#SIMT_Stack). For
    comparison, I discussed branch divergence in
    [this post]({{ site.baseurl }}{% post_url 2014-12-06-branch-divergence-in-parallel-kernels %}),
     but in the context of AMD's Southern Islands instruction set, which uses
     the 'exec' register rather than a stack to control execution.
  * Operand collector. Like many GPUs, GPGPU-sim simulates banked register
    files to save area and power (I described VideoCore's approach in [this
    post]({{ site.baseurl }}{% post_url 2016-03-01-videocore-qpu-pipeline %})).
    However, it uses a clever optimization to dynamically schedule register
    file access so the compiler doesn't need to worry about it. This
    implementation is described in detail
    [here](http://gpgpu-sim.org/manual/index.php/Main_Page#Register_Access_and_t
    he_Operand_Collector).
  * Scoreboard. GPGPU-sim uses a scoreboard to track register dependencies, as
    described
    [here](http://gpgpu-sim.org/manual/index.php/Main_Page#Scoreboard). Nyuzi
    uses a similar scheme, as I documented in
    [this post]({{ site.baseurl }}{% post_url 2014-05-27-keeping-score %}).
  * Memory pipeline. This includes an address generation unit and access coalescing.

### MIAOW

[https://github.com/VerticalResearchGroup/miaow](https://github.com/VerticalResearchGroup/miaow)
[http://pages.cs.wisc.edu/~vinay/pubs/MIAOW-coolchips-paper.pdf](http://pages.cs.wisc.edu/~vinay/pubs/MIAOW-coolchips-paper.pdf)

This is a newer project, but is interesting in that it is binary compatible
with a modern commercial (Southern Islands) GPU instruction set. Like GPGPU-
Sim, it is only the compute unit and has no fixed function graphical units, so
it is unable to simulate graphics rendering. However, unlike GPGPU-sim, it is
a synthesizable hardware implementation. As stated in the Wiki: _"A primary
motivator for MIAOW's creation is the belief that software simulators of
hardware such as CPUs and GPUs often miss many subtle aspects that can skew
the performance, power, and other quantitative results that they produce."_
Creating synthesizable hardware ensures they accurately model hardware
tradeoffs for their design decisions and can't unwittingly "cheat" like
software simulators can.
