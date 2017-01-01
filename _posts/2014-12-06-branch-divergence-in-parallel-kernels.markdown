---
layout: post
title: Branch Divergence
date: '2014-12-06T23:06:00.000-08:00'
author: Jeff
tags:
- amd
- branch divergence
- southern islands
- llvm
modified_time: '2016-11-05T06:46:13.485-07:00'
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-3477634446071970093
blogger_orig_url: http://latchup.blogspot.com/2014/12/branch-divergence-in-parallel-kernels.html
---

Modern GPUs utilize [SIMD](http://en.wikipedia.org/wiki/SIMD) heavily (under
various monikers: SPMD, SIMT, etc).  Whatever the name, it fundamentally boils
down to duplicating arithmetic units and applying the same instruction to many
vector lanes, each of which represents an independent instance of the program.
Because the non-arithmetic parts of the pipeline are shared, this allows
packing more computation in a smaller amount of space.

This works well until conditional execution is thrown into the mix.  It's
possible (common, actually) that some of the instances will chose one branch
and some will chose another.  This is called "branch divergence," and is an
active area of research. I've recently written a
[compiler](https://github.com/jbush001/NyuziToolchain/tree/master/tools/spmd-
compile) for a simple, C-like language that can produce parallel kernels for
the instruction set of the
[processor](https://github.com/jbush001/NyuziProcessor) I've been working on.
It uses the LLVM backend I've already developed for this architecture. It's
interesting to compare the generated code to a modern GPU, AMD's "Southern
Islands" architecture.

The [reference guide](http://developer.amd.com/wordpress/media/2012/12/AMD_Sou
thern_Islands_Instruction_Set_Architecture.pdf) for AMD's Southern Islands
architecture, gives an example of a kernel that can exhibit divergent
execution in section 2.4:

{% highlight c %}
{% raw %}
float fn0(float a, float b)
{
    if (a > b)
        return a * a - b;
    else
        return b * b - a;
}
{% endraw %}
{% endhighlight %}

They also include the generated assembly code for this. It's fairly
understandable if you know a few basic things:

  * Comparisons set a vector condition code register (which stores a separate
    result for each lane). This is exposed through a register called 'vcc',
    which can be used like any other general purpose register.
  * There is a special register called 'exec' that controls which vector lanes
    a result will be written back to. It tracks which threads are active on the
    current code path.

Here is their code:

         v_cmp_gt_f32 r0, r1        // a>b
         s_mov_b64 s0, exec         // Save current exec mask
         s_and_b64 exec, vcc, exec  // Do if
         s_cbranch_vccz label0      // Branch if all lanes fail
         v_mul_f32 r2, r0, r0       // result = a * a
         v_sub_f32 r2, r2, r1       // result = result - b
    label0:
         s_not_b64 exec, exec       // Do else
         s_and_b64 exec, s0, exec   // Do else
         s_cbranch_execz label1     // Branch if all lanes fail
         v_mul_f32 r2, r1, r1       // result = b * b
         v_sub_f32 r2, r2, r0       // result = result - a
    label1:
         s_mov_b64 exec, s0         // Restore exec mask

On the third line, the result of the comparison in vcc is moved into exec.
From this point on, for lanes that the condition was not true for, there will
be no effect. Results will be computed, but the registers will not be updated.
After label0, the current mask is inverted, then combined with the previous
mask (this would properly handle the case of nested masks).

*As an aside, previous versions of AMD's architecture handled masks
automatically with a hardware stack. For example, the R700 had PUSH,
PUSH_ELSE, and POP instructions. In the newer architectures, they apparently
have moved more of this management to the compiler, which must allocate
registers to hold intermediate masks when dealing with nested conditionals.*

I'm going to back up a bit and talk about how the equivalent code is generated
for my architecture from the same kernel source code given above. The compiler
I've written starts out by creating an [abstract syntax
tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) for the parsed
source. It then walks the tree recursively, calling the generate() method on
each node, which then call into a class called SPMDBuilder to create LLVM
intermediate code as side effect.

{% highlight c++ %}
{% raw %}
class IfAst : public AstNode {
public:
...
    virtual llvm::Value *generate(SPMDBuilder&);

private:
  AstNode *Cond;
  AstNode *Then;
  AstNode *Else;
};

Value *IfAst::generate(SPMDBuilder &Builder)
{
  Builder.pushMask(Cond->generate(Builder));
  Then->generate(Builder);
  if (Else) {
    Builder.invertLastPushedMask();
    Else->generate(Builder);
  }

  Builder.popMask();
  return nullptr;
}
{% endraw %}
{% endhighlight %}

SPMDBuilder (which stands for Single Program Multiple Data, the programming
model we are using) is a wrapper I've added around the LLVM's standard
[IRBuilder](http://llvm.org/docs/doxygen/html/classllvm_1_1IRBuilder.html)
class, which constructs the [intermediate
representation](http://llvm.org/docs/LangRef.html). SPMDBuilder adds a few
special new behaviors. The first is pushMask(), which takes the result from a
vector comparison (the Cond variable in this case) and causes it to be applied
to subsequent operations. Masks are stored in an internal stack within the
SPMDBuilder, and are pushed and popped as each nested conditional clause is
generated. Note also invertLastPushedMask in the else clause.

*(for simplicity, I'm not generating short circuit jumps here if all of the
conditions go the same way like the AMD code above. It's not hard to do. There
is already a function in SPMDBuilder called shortCircuitZeroMask to handle
this. From a performance perspective, it is debatable. It some cases it is
faster to simply execute the instructions than take the hit of pipeline flush
that a branch would produce. Of course, I should measure that :).*

Unlike AMD SI architecture, there isn't an global exec register in this
architecture. Each instruction can optionally take a mask register as a
parameter. SPMDBuilder takes care of applying this mask automatically. It only
need be applied when a value is written back to a local variable.  The meat of
that is here, in assignLocalVariable:

{% highlight c++ %}
{% raw %}
void SPMDBuilder::assignLocalVariable(Value *Variable, Value *NewValue)
{
  if (MaskStack.empty()) {
    Builder.CreateStore(NewValue, Variable);
  } else {
    // Need to predicate this instruction
    llvm::Function *BlendFunc = llvm::Intrinsic::getDeclaration(MainModule,
                           (llvm::Intrinsic::ID) Intrinsic::nyuzi_vector_mixf,
                           None);

    Value *OldValue = Builder.CreateLoad(Variable);

    SmallVector Ops;
    Ops.push_back(getCurrentMask());
    Ops.push_back(NewValue);
    Ops.push_back(OldValue);

    Value *Blended = Builder.CreateCall(BlendFunc, Ops, "");
    Builder.CreateStore(Blended, Variable);
  }
}
{% endraw %}
{% endhighlight %}

This is a little hard to follow, but the important part is the call to
nyuzi_vector_mixf. Although this looks like a function call, it is actually an
*intrinsic* that is specific to this architecture. Intrinsic functions are a
way of extending LLVM without needing to add new LLVM instructions to the
intermediate representation. They look like function calls on the front end,
but they have special properties that can be used during instruction
selection. Here is the LLVM intermediate code generated by above:

    %2 = call i32 @llvm.nyuzi.__builtin_nyuzi_mask_cmpf_gt(<16 x float> %0,
         <16 x float> %1)
    ...
    %5 = fmul <16 x float> %3, %4
    %7 = fsub <16 x float> %5, %6
    %8 = load <16 x float>* %result
    %9 = call <16 x float> @llvm.nyuzi.__builtin_nyuzi_vector_mixf(i32 %2,
         <16 x float> %7, <16 x float> %8)

The 16 bit mask, which is the result of the comparison, is the first parameter
to the mixf intrinsic. The second is the newly computed value from fsub, and
the third is the old value in the variable.  For each bit in the mask, it
selects the lane from the first parameter if the bit is 1, and the second if
it is zero (which is just the value it had before).  The fact that it takes
the old value as a parameter may seem a bit strange, but bear in mind that
LLVM intermediate code is Static Single Assignment form (SSA), so we can never
assign the same variable twice.

_As an aside, you may have noticed that the code above generates stores and
loads for every local variable access.  The [mem2reg
pass](http://llvm.org/docs/Passes.html#mem2reg-promote-memory-to-register) in
LLVM will convert these to use registers as appropriate. This is the
officially recommended way to do this: it saves having to generate [phi
nodes](http://llvm.org/docs/LangRef.html#phi-instruction) in the front end and
is simpler._

__builtin_nyuzi_vector_mixf  doesn't correspond to an instruction in this
architecture and has no meaning to the LLVM infrastructure. It gets passed
through all the way through various transformation passes until instruction
selection. Here is where the magic happens.  This is NyuziInstrInfo.td, the
instruction generation template for this architecture, which is used by
[TableGen](http://llvm.org/docs/TableGen/index.html).

    let Constraints = "$dest = $oldvalue" in {
        // Vector = Vector op Vector, masked
        def VVVM : FormatRMaskedTwoOpInst<
            (outs VR512:$dest),
            (ins GPR32:$mask, VR512:$src1, VR512:$src2, VR512:$oldvalue),
            operator # "_mask $dest, $mask, $src1, $src2",
            [(set v16i32:$dest, (int_nyuzi_vector_mixi i32:$mask,
                 (OpNode v16i32:$src1, v16i32:$src2), v16i32:$oldvalue))],
            opcode,
            FmtR_VVVM>;
    }

The seventh line (with int_nyuzi_vector_mixi) is the pattern that is matched
during instruction selection. The intermediate instruction representation of
LLVM is stored as a directed acyclic graph. This pattern represents a fragment
of a DAG structure that this instruction will be emitted for.  This allows the
compiler to combine several LLVM IR instructions into a single target
instruction, sub_f_mask, which you can see in the resulting Nyuzi assembly
code generated by the compiler:

     cmpgt_f s1, v0, v1          // a>b
     and s2, s1, s0              // and result with previous mask
     mul_f v2, v0, v0            // a*a
     sub_f_mask v2, s2, v2, v1   // result = a*a-b (for active kernels)
     xor s1, s1, -1              // invert the comparison mask
     and s1, s1, s0              // recombine with previous mask
     mul_f v1, v1, v1            // b*b
     sub_f_mask v2, s1, v1, v0   // result = b*b-a (for active kernels)

In this code, s0 is the old mask before this conditional code is executed, and
s1 is the mask after the condition has been applied.  The subtract
instructions apply the mask, but the multiplications (which are only
intermediate results, stored in temporary registers) don't bother to.  In the
middle of this code, the xor instruction inverts the mask, which is then anded
with the previous mask in the mask stack.
