---
layout: post
title: Lua Closure Walkthough
date: '2017-01-11T18:00:00.000-08:00'
author: Jeff
tags:
- lua
- closure
---

In the [last post]({{ site.baseurl }}{% post_url 2017-01-10-closures %}) I described
a challenge with implementing closures that capture by reference.

The [Lua](https://www.lua.org/) 5.0 VM has an interesting solution to this problem,
described in section 5 of [this document](https://www.lua.org/doc/jucs05.pdf).
The inner function always accesses free variables (which they refer to as
'outer local variables') indirectly through a structure called an 'upval' using
the instructions GETUPVAL and SETUPVAL. While the enclosing function is active,
the upval points to the local variable slot on the stack (though Lua is a
register based VM, it uses a stack internally) This is handy because the
enclosing function doesn't need to worry about whether the variable is part of
a closure or not. When the function returns, it copies local variables into a
member of the upval structure and makes the upval point to that. The upval
sticks around until it is no longer referenced by anyone, at which point it can
be garbage collected.

Let's walk through an example:

{% highlight lua linenos %}
function make_closure()
    local i = 1
    local func = function()
        print(i)
    end

    i = i + 1
    return func
end

func = make_closure()
func()
{% endhighlight %}

When I run this, I get the following:

    $ lua closure.lua
    2

Even though it updates 'i' after it creates the closure, the inner function sees the
new version when it executes.

I can dump the virtual machine code for this:

    $ luac -l closure.lua

    function <closure.lua:1,9> (6 instructions, 24 bytes at 0x7fdbf6c03960)
    0 params, 2 slots, 0 upvalues, 2 locals, 1 constant, 1 function
    	1	[2]	LOADK    	0 -1	; 1
    	2	[5]	CLOSURE  	1 0	; 0x7fdbf6c03c20
    	3	[5]	MOVE     	0 0
    	4	[7]	ADD      	0 0 -1	; - 1
    	5	[8]	RETURN   	1 2

    function <closure.lua:3,5> (4 instructions, 16 bytes at 0x7fdbf6c03c20)
    0 params, 2 slots, 1 upvalue, 0 locals, 1 constant, 0 functions
    	1	[4]	GETGLOBAL	0 -1	; print
    	2	[4]	GETUPVAL 	1 0	; i
    	3	[4]	CALL     	0 2 1
    	4	[5]	RETURN   	0 1

I'll walk through these an instruction at a time. The first function in this listing
is make_closure.

{% highlight lua %}
local i = 1
{% endhighlight %}

    1	[2]	LOADK    	0 -1	; 1

LOADK loads a constant value. Lua is a virtual register machine. the first parameter
is a virtual register number (0), which represents the local variable i. The second
is an index into the constant table (it displays indices as negative numbers in the
listings to differentiate them from registers)

Next it creates the closure

{% highlight lua %}
    local func = function()
        print(i)
    end
{% endhighlight %}

    2	[5]	CLOSURE  	1 0	; 0x7fb49ac03c20
    3	[5]	MOVE     	0 0

The closure instruction allocates a closure object that references the inner
function. The inner function is not emitted here: it is a separate function
that this code references. The second parameter indexes into a table of
closures referenced by this function (there's only closure, so it's index 0). The
MOVE instruction after this is not executed. The Lua interpreter needs a place
to stash the list of local variables that the closure references. It inserts
'pseudo-instructions' that have the register indices. The CLOSURE instruction
skips these and execution resumes after them.

Looking at the Lua interpreter code,
[lvm.c](https://github.com/LuaDist/lua/blob/d2e7e7d4d43ff9068b279a617c5b2ca2c2771676/src/lvm.c#L642):

{% highlight c linenos %}
{% raw %}
      case OP_CLOSURE: {
        Proto *p;
        Closure *ncl;
        int nup, j;
        p = cl->p->p[GETARG_Bx(i)];
        nup = p->nups;
        ncl = luaF_newLclosure(L, nup, cl->env);
        ncl->l.p = p;
        for (j=0; j<nup; j++, pc++) {
          if (GET_OPCODE(*pc) == OP_GETUPVAL)
            ncl->l.upvals[j] = cl->upvals[GETARG_B(*pc)];
          else {
            lua_assert(GET_OPCODE(*pc) == OP_MOVE);
            ncl->l.upvals[j] = luaF_findupval(L, base + GETARG_B(*pc));
          }
        }
        setclvalue(L, ra, ncl);
        Protect(luaC_checkGC(L));
        continue;
      }
{% endraw %}
{% endhighlight %}

As mentioned above, in line 5, the B argument indexes into a table
of closures, which is of type 'Proto'. This structure has the number of upvals
(nups). The loop (line 9) reads the opcodes of following pseudo-instructions (The
'pc++' in the loop increment advances past the instruction). If the opcode is
OP_GETUPVAL, it uses an existing upval from another closure. If it is OP_MOVE
(as it is in this example), it calls luaF_findupval.


luaF_findupval is defined in
[lfunc.c](https://github.com/LuaDist/lua/blob/d2e7e7d4d43ff9068b279a617c5b2ca2c2771676/src/lfunc.c#L96).

{% highlight c linenos %}
{% raw %}
UpVal *luaF_findupval (lua_State *L, StkId level) {
  global_State *g = G(L);
  GCObject **pp = &L->openupval;
  UpVal *p;
  UpVal *uv;
  while (*pp != NULL && (p = ngcotouv(*pp))->v >= level) {
    lua_assert(p->v != &p->u.value);
    if (p->v == level) {  /* found a corresponding upvalue? */
      if (isdead(g, obj2gco(p)))  /* is it dead? */
        changewhite(obj2gco(p));  /* ressurect it */
      return p;
    }
    pp = &p->next;
  }
  uv = luaM_new(L, UpVal);  /* not found: create a new one */
  uv->tt = LUA_TUPVAL;
  uv->marked = luaC_white(g);
  uv->v = level;  /* current value lives in the stack */
  uv->next = *pp;  /* chain it in the proper position */
  *pp = obj2gco(uv);
  uv->u.l.prev = &g->uvhead;  /* double link it in `uvhead' list */
  uv->u.l.next = g->uvhead.u.l.next;
  uv->u.l.next->u.l.prev = uv;
  g->uvhead.u.l.next = uv;
  lua_assert(uv->u.l.next->u.l.prev == uv && uv->u.l.prev->u.l.next == uv);
  return uv;
}
{% endraw %}
{% endhighlight %}

This checks to see if there is already an upval for this local variable (line 6-14).
This would happen if code before the current instruction had referenced it. If so,
it returns the existing one so all closures share it. If not, it creates a new one and adds
it to a list 'L->openupval' associated with the function (line 3, 19). It also makes the
pointer uv->v point to the local variable (passed in 'level'), line 18 which in the
previous listing (line 14) was set to the stack slot (base + GETARG_B(*pc))

Now it adds a value to the local variable i.

{% highlight lua %}
    i = i + 1
{% endhighlight %}

    	6	[8]	ADD      	0 0 -1	; - 1

As before, the -1 refers to the first entry in the constant table, which is 1
(instruction parameters have 8 bits. Values 0-127 are register values, 128-255
refer to constant table entries, which this treats as signed bytes). This is
equivalent to r0 = r0 + 1. Note that this accesses 'i' as a local variable. The
upval is still pointing to the local register slot.

Finally it returns the closure object.

{% highlight lua %}
    return func
{% endhighlight %}

    	7	[9]	RETURN   	1 2

The local register slot goes away at this point. The RETURN instruction has a
side effect of modifying the upvals to point to themselves before the function
exits. Looking at the Lua interpreter code
([lvm.c](https://github.com/LuaDist/lua/blob/d2e7e7d4d43ff9068b279a617c5b2ca2c2771676/src/lvm.c#L642)).


{% highlight c linenos %}
{% raw %}
   case OP_RETURN: {
     int b = GETARG_B(i);
     if (b != 0) L->top = ra+b-1;
     if (L->openupval) luaF_close(L, base);
     L->savedpc = pc;
{% endraw %}
{% endhighlight %}

If there are items in the L->openupval list (line 4), it calls luaF_close :

{% highlight c linenos %}
{% raw %}
 void luaF_close (lua_State *L, StkId level) {
   UpVal *uv;
   global_State *g = G(L);
   while (L->openupval != NULL && (uv = ngcotouv(L->openupval))->v >= level) {
     GCObject *o = obj2gco(uv);
     lua_assert(!isblack(o) && uv->v != &uv->u.value);
     L->openupval = uv->next;  /* remove from `open' list */
     if (isdead(g, o))
       luaF_freeupval(L, uv);  /* free upvalue */
     else {
       unlinkupval(uv);
       setobj(L, &uv->u.value, uv->v);
       uv->v = &uv->u.value;  /* now current value lives here */
       luaC_linkupval(L, uv);  /* link upvalue into `gcroot' list */
     }
   }
 }
{% endraw %}
{% endhighlight %}

The luaF_close function walks through open upvals in the list L->openupval
(which it added the upval to in luaF_findupval). In lines 12 & 13 it copies
the value from the upval data pointer (v) into the upval object (u.value) and
then sets the upval pointer to itself.

Let's look now at the inner function:

    function <closure.lua:4,6> (4 instructions, 16 bytes at 0x7fca73403c20)
    0 params, 2 slots, 1 upvalue, 0 locals, 1 constant, 0 functions
    	1	[5]	GETGLOBAL	0 -1	; print
    	2	[5]	GETUPVAL 	1 0	; i
    	3	[5]	CALL     	0 2 1
    	4	[6]	RETURN   	0 1

Instruction 2 uses GETUPVAL to reads the 'i' variable from the upval into a
temporary local register 1. Here's the code in
([lvm.c](https://github.com/LuaDist/lua/blob/d2e7e7d4d43ff9068b279a617c5b2ca2c2771676/src/lvm.c#L642)):

{% highlight c linenos %}
{% raw %}
      case OP_GETUPVAL: {
        int b = GETARG_B(i);
        setobj2s(L, ra, cl->upvals[b]->v);
        continue;
      }
{% endraw %}
{% endhighlight %}

The B argument indexes into the upvals array and get the reference v. The setobj2s copies
the value from the pointer into the result register (called ra in this code).

Finally, instruction 3 called the 'print' function to output the value, then the function
returns.

This is a clever solution that simplifies the implementation of the compiler. One approach
to supporting by-reference closures in my system would be to use a 'cons' object in a similar
way that the upval structure is used here.
