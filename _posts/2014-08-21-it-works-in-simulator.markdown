---
layout: post
title: '"It Works in the Simulator..."'
date: '2014-08-21T22:04:00.000-07:00'
author: Jeff
tags:
- simulation
- synthesis
- FPGA
- SystemVerilog
modified_time: '2016-04-11T18:38:48.840-07:00'
thumbnail: https://img.youtube.com/vi/7K8E7d01fL0/default.jpg
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-5521202908785581076
blogger_orig_url: http://latchup.blogspot.com/2014/08/it-works-in-simulator.html
---

I do most hardware development in simulation. It allows a level of visibility
that software developers can only dream of: I can see the state of every
signal at every instant for the entire run. But there is a dark side to
simulation: the synthesized design may not behave the same as the simulation.
There are a few well known [patterns](http://www.eda.org/vlog-
synth/Mills_Final.pdf) that lead to synthesis/simulation mismatches that are
easy enough to avoid with attentive coding, but the existence of these
gremlins always leaves me uneasy...

I recently began bringup the next generation of my [parallel
processor](https://github.com/jbush001/NyuziProcessor) on FPGA.  I'd shaken a
lot of bugs out in simulation using randomized instruction streams and
directed test programs.  I fixed some nit-picky synthesis errors and loaded it
onto the board and... nothing.

In a situation like this, there are a few approaches I use.  First, I routed
LEDs to display internal signals. The program counter was advancing, so it did
appear to be executing code, but that was all I could tell using that
debugging method. I had hacked together a simple embedded logic analyzer,
which captures internal signals and dumps them out the serial port when
triggered, which allowed me to get a more detailed view of what was going on.
Iteration was slow, because it takes over an hour to synthesize the design in
my VirtualBox Linux instance and I don't have a lot of time each day to spend
on this, but after a few weeks, I finally isolated three major problems and
got the design running pretty well.

The first issue had to do with port direction specifiers: I had marked as an
output port something that should be an input through sloppy copy/pasting. It
seems like this should immediately raise an error, but the way the Verilog
language is designed makes this less clear cut than one might expect. It is
perfectly valid to read output signals within a module. It's also okay to
connect multiple outputs together, because some of them may be driving it with
the Z (high impedance) state. At any rate, Verilator was perfectly happy with
it and it simulated fine.  However, Quartus, seeing that the output port had
no driver within that module, optimized away the logic entirely (to be fair,
it did print a warning)

The second issue had to do with the way Altera's built-in SRAM "Megafunction"
(ALTSYNCRAM) behaves when the same address is read and written on different
ports during the same cycle. This design expects that the read will return the
data that is written on the other port.  Altera's documentation describes a
parameter that controls how it should behave.  I set the value "NEW_DATA" as
suggested.  However, as it turns out, my particular FPGA model doesn't support
that value.  Quartus didn't raise a warning, it just ignored the parameter and
returned the old data at that address.  Adding bypass logic in my own design
remedied the issue.

But the last one was the most nasty.  I looked at this code many times and
didn't spot it.  Can you?

        assert(dd_instruction.has_dest && !dd_instruction.dest_is_vector)
        wb_writeback_value <= sq_store_sync_success;

The problem is that a semicolon is missing from the end of the assert
statement.  The [language
spec](http://standards.ieee.org/getieee/1800/download/1800-2012.pdf) says that
assert() basically works like an if statement.  If the condition in
parentheses is true, the following statement is executed.  It can even have an
else statement. I knew this, but I just overlooked the fact that the semicolon
was missing, and hadn't appreciated what nasty bugs could be introduced if it
was inadvertently omitted.

In simulation, leaving the semicolon off doesn't have change program behavior.
Since the assertion value is always true (as it's supposed to be) the
nonblocking assignment executes.  However, during synthesis, Quartus treats
the assignment as part of the assertion, which is _not_ synthesizable, and
removes it entirely.  It doesn't give a warning as this is expected behavior.

In the end, these turned out to be fairly tractable, and I learned some new
things to watch out for when I do see issues.

Anyway, here's a single core clocked at 50 Mhz rendering a Phong shaded teapot
with 2300 triangles:

<iframe allowfullscreen="" frameborder="0" height="315" src="https://www.youtube.com/embed/7K8E7d01fL0" width="560"></iframe>

That's around 2 frames per second.  It's not setting any speed records yet,
but it works! :)
