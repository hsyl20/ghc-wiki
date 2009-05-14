# GHC Backend Ideas



This page is a collection of ideas for improving GHC's back-end, from
codeGen downwards.



Some of these ideas have been kicking around for a while, some are the
result of a discussion between myself (Simon M.) and Roman
Leshchinskiy today (2006-05-26).


## Overview



The general direction we want to move in is to drop the use of gcc as
an optimising code generator in favour of the native code generator
and C--.  The reasons for dropping gcc are many, briefly:


- Complexity: the mangler, enough said
- Fragility: new versions of gcc usually break GHC, bugs occur
  that are dependent on the gcc version, gcc is one more variable
  in a bug report.
- Compilation speed: gcc keeps getting slower, -fvia-C is already
  more that 2x slower than -fasm.
- Flexibility: we want more freedom to define our own calling conventions
  and register assignments.  If we drop the requirement for -fvia-C,
  we can improve performance (see below).
- Dependencies: we don't want to ship gcc with GHC on Windows.


Ultimately C-- should provide the best code generation route in terms
of portability and efficiency, but the compiler isn't there yet, and
C-- code generation in GHC is dependent on some major refactoring in
codeGen (we need to decouple the explicit stack handling from the rest
of the code generator).



Note that we still want to retain the unregisterised C route.  It is
essential for bootstrapping, and it doesn't suffer from the problems
above: we don't care about peformance, and unreg doesn't require the
mangler.


## Improving code for loops



The NCG already generates reasonable code compared to -fvia-C, but
that's only because -fvia-C isn't particularly good.  We have known
for a long time that we don't give gcc enough room to really optimise the
code well, especially loops.  Historically this hasn't been a problem,
because most Haskell programs don't contain small tight loops.  Now
however, we're starting to see more code that relies on good loop
performance (Data.ByteString and Data.Array.Parallel).



The code generated by GHC for a tail-recursive loop uses an explicit
tail call for the recursion, which gcc cannot spot as a branch.
Furthermore, arguments to the recursive function are passed either in
fixed registers or on the stack.


### Interim solution



As a first step on the way to better code, we can do a Cmm to Cmm
optimisation which reconstructs loops using branching and with
loop-carried data in local variables.  (Roman has implemented this,
and plans to clean up/commit shortly).  This is partly a temporary
solution, long term we plan to improve the native code generator with
real loop optimisations, described below.


### Heap/stack checks



We often see tail-recursive functions that include a heap or stack
check purely for the exit case of the loop.  See [\#1498](http://gitlabghc.nibbler/ghc/ghc/issues/1498). For example, if the
function looks something like this:


```wiki
f = \x y -> case x of
             0 -> g (Just y)
             _ -> f (x+1) y
```


The allocation in the 0 branch of the case causes a heap check at the
beginning of f, and an Hp subtraction before the recursive call to f.



We should only do the heap check once in this case.  Either: 


- aggregate the heap check to the caller.  Difficult if f is
  exported.

- generate a worker/wrapper pair, where the wrapper does the heap
  check.  eg.

```wiki
 f:  
   Hp += 3;
   if (Hp >= HpLim) { ... }
  L:
   ... recursive call jumps to L ...
```

- Generate the above code with a special-case Cmm to Cmm optimisation that
  spots the heap check.

- Let some general-purpose heavy-duty loop optimisation transform
  into the above code (apparently gcc can do this).

## Framework for branch prediction



On modern CPUs correctly predicted branches are nearly free while mis-predicted branches incurr quite a high penalty due to pipeline stalls etc.



There are a number of low level things that can be done if we have some idea of the probability of a branch being taken. For example the block corresponding to the branch that is unlikely to be taken can be made the target of the jump so that the fall through branch is the likely one. On some CPUs its possible to add explicit hints to conditional jump instructions. If a branch is comparatively very unlikely then its block can be moved completely out of line. See a CPU architecture/optimisation guide for more suggestions.



GCC has a framework for gathering and using branch prediction information to improve code generation. It can get information from a static analysis and heuristics, from profile feedback and ffrom explicit user annotations via builtin\_expect().



So the suggestion is that GHC could have a similar framework including explicit user annotations. It is believed that this could make a significant difference to the speed of some low level code like ByteString.



See ticket [\#849](http://gitlabghc.nibbler/ghc/ghc/issues/849) for more details.


## Improving and refactoring the native code generator


### Allow SSA to be expressed in Cmm



Cmm can almost express SSA, but it lacks phi.  Adding phi to Cmm
wouldn't be difficult, and then we can express real loops and
implement the well-known loop optimisations.


### Implement loop optimisations



Add loop optimisations as Cmm to Cmm passes.


### Clean up the NCG



The NCG is currently structured as follows: the "instruction selector"
(`MachCodeGen`) takes Cmm and generates `[Instr]`, where
`Instr` is a machine-dependent instruction datatype.  Register
allocation is then done on `[Instr]`, the register allocator is
machine-independent but relies on some machine-dependent functions
over `Instr`.



The problem with this is that we can't express any generic
optimisations at the instruction level, and we can't share any code
between the instruction selectors.  Also, the machine-dependent parts
of the register allocator duplicate a lot of code.



A better approach might be for the machine-dependent instruction
selector to generate Cmm instead; it would be a stylised form of Cmm
in which every statement corresponds to a single instruction in the
target assembly language, such that assembly can be generated by
pretty-printing this Cmm (call it MD-Cmm) directly.



This might entail encoding some instructions as function calls, or
extending the range of MachOps to accomodate operations supported by
some processors.  Also, we probably want to abstract Cmm over the
register type (MD-Cmm probably has "register classes", to indicate
that a particular register must be drawn from a certain subset of
machine registers).



Register allocation would be done on MD-Cmm, using information only
about the registers and register classes of the target architecture.
There would be less machine dependent code here.



All in all, this should result in a reduction in code size of the NCG
and improvements in maintainability and portability.  Longer-term it
will enable us to have MD-Cmm to MD-Cmm passes to do things like
instruction scheduling.



(note that the general idea for doing this comes from gcc, we are
using MD-Cmm where they use RTL).



There's even a nice migration path here.  We don't need to transition
every architecture to MD-Cmm right away, we can implement an MD-Cmm
route for a single architecture, because the existing register
allocator can be made to work over MD-Cmm by providing appropriate
bits specific to MD-Cmm, I believe.  Roman plans to do this (in his
spare time :-) for Sparc.


### Aliasing information



We will need at some point access to aliasing information in Cmm, for
the Cmm-to-Cmm passes to make use of.


## Improving register usage


### Re-using fixed registers



The register allocator should treat fixed registers (like the heap
pointer) in the same way as other registers: they can be spilled and
reloaded.  This would let an inner loop that doesn't need access to Hp
re-use the register, for example.


### Eliminating BaseReg



If the current capability was stored on the C stack, then we wouldn't
need a separate BaseReg register, freeing up a register (especially
handy on x86).  I don't see how to do this if we compile via C, but if
we go NCG-only it would be possible.



We could either do this by copying the capability onto the stack
before running Haskell code (and be careful that the copy doesn't get
out of sync with the real thing), or we could allocate some space
below each Capability and point the stack pointer directly at it
before entering Haskell.

