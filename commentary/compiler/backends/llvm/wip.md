


# Work in Progress on the LLVM Backend



This page is meant to collect together information about people working on (or interested in working on) LLVM in GHC, and the projects they are looking at.  See also the [state of play of the whole back end](commentary/compiler/new-code-gen). This is more a page of ideas for improvements to the LLVM backend and less so an indication of actual work going on.


### LLVM IR Representation



The LLVM IR is modeled in GHC using an algebraic data type to represent the first order abstract syntax of the LLVM assembly code. The LLVM representation lives in the 'Llvm' subdirectory and also contains code for pretty printing. This is the same approach taken by  EHC's LLVM Back-end, and we adapted the  module developed by them for this purpose. 



The current design is overly complicated and could be faster. It uses String + show operations for printing for example when it should be using FastString + Outputable. Before simplifying this design though it would be good to investigate using the LLVM API instead of the assembly language for interacting with LLVM. This would be done most likely by using the pre-existing Haskell LLVM API bindings found [
here](http://hackage.haskell.org/package/llvm). This should hopefully provide a speed up in compilation speeds which is greatly needed since the LLVM back-end is \~2x slower at the moment.


### TABLES\_NEXT\_TO\_CODE



We now support [TNTC](commentary/compiler/backends/llvm/issues#) using an approach of gnu as subsections. This seems to work fine but we would like still to move to a pure LLVM solution. Ideally we would implement this in LLVM by allowing a global variable to be associated with a function, so that LLVM is aware that the two will be laid out next to each other and can better optimise (e.g using this approach LLVM should be able to perform constant propagation on info-tables).



**Update (30/06/2010):** The current TNTC solution doesn't work on Mac OS X. So we need to implement an LLVM based solution. We currently support OS X by post processing the assembly. Pure LLVM is a nicer way forward.


### LLVM Alias Analysis Pass



**Update: This has been implemented, needs more work though**



LLVM doesn't seem to do a very good job of figuring out what can alias what in the code generated by GHC. We should write our own alias analysis pass to fix this.


### Optimise LLVM for the type of Code GHC produces



At the moment only a some fairly basic benchmarking has been done of the LLVM back-end. Enough to give an indication of how it performs on the whole (well as far as you trust benchmarks anyway) and of what it can sometimes achieve. However this is by no means exauhstive or probably even close to it and doesn't give us enough information about the areas where LLVM performs badly. The LLVM optimisation pass also at the moment just uses the standard '-O\[123\]' levels, which like GCC entail a whole bunch of optimisation passes. These groups are designed for C programs mostly.



So:


- More benchmarking, particularly finding some bad spots for the LLVM back-end and generating a good picture of the characteristics of the back-end.
- Look into the LLVM optimiser, e.g perhaps some more work in the style of [
  Don's work](http://donsbot.wordpress.com/2010/03/01/evolving-faster-haskell-programs-now-with-llvm/)
- Look at any new optimisation passes that could be written for LLVM which would help to improve the code it generates for GHC.
- Look at general fixes/improvement to LLVM to improve the code it generates for LLVM.
- Sometimes there is a benefit from running the LLVM optimiser twice of the code (e.g opt -O3 \| opt -O3 ...). We should add a command line flag that allows you to specify the number of iterations you want the LLVM optimiser to do.

### Update the Back-end to use the new Cmm data types / New Code Generator



There is ongoing work to produce a new, nicer, more modular code generator for GHC (the slightly confusingly name code generator in GHC refers to the pipeline stage where the Core IR is compiled to the Cmm IR). The LLVM back-end could be updated to make sure it works with the new code generator and does so in an efficient manner.


### LLVM's Link Time Optimisations



One of LLVM's big marketing features is its support for link time optimisation. This does things such as in-lining across module boundaries, more aggressive dead code elimination... etc). The LLVM back-end could be updated to make use of this. Roman apparently tried to use the new 'gold' linker with GHC and it doesn't support all the needed features.


- [
  http://llvm.org/releases/2.6/docs/LinkTimeOptimization.html](http://llvm.org/releases/2.6/docs/LinkTimeOptimization.html)
- [ http://llvm.org/docs/GoldPlugin.html](http://llvm.org/docs/GoldPlugin.html)

### LLVM Cross Compiler / Port



This is more of an experimental idea but the LLVM back-end looks like it would make a great choice for Porting LLVM. That is, instead of porting LLVM through the usual route of via-C and then fixing up the NCG, just try to do it all through the LLVM back-end. As LLVM is quite portable and supported on more platforms then GHC, it would be an interesting and valuable experiment to try to port GHC to a new platform by simply getting the LLVM back-end working on it. (The LLVM back-end works in both unregistered and registered mode, another advantage for porting compared to the C and NCG back-ends).



It would also be interesting to looking into improving GHC to support cross compiling and doing this through the LLVM back-end as it should be easier to fix up to support this feature than the C or NCG back-ends.


### Get rid of Proc Point Splitting



When Cmm code is first generated a single Haskell function will be mostly compiled to one Cmm function. This Cmm function isn't passed to the backends though as the CPS style used in it requires that the backends be able to take the address of labels in a function since they're used as return points. The C backend can't support this. While there is a GNU C extension allowing the address of a label to be taken, the address can only be used locally (in the same function). So what proc point splitting does is cut a single Cmm function into multiple top level Cmm functions so that instead of needing to take the address of a label, we now take the address of a function.



It would be nice to get rid of proc point splitting. This is one of the goals for the new code generator. This will give us much bigger Cmm functions which should give more room for LLVM to optimise. There is an issue though that LLVM doesn't support taking the address of a local label either. So will need to add support to LLVM for taking label addresses or convert CPS style into something more direct if thats possible.


### Don't Pass Around Dead STG Registers



**Update: This has been implemented**



At the moment in the LLVM backend we always pass around the pinned STG registers as arguments for every Cmm function. A huge amount of the time though we aren't storing anything in the STG registers, they are dead really. If we can treat the correctly as dead then LLVM will have more free registers and the allocator should do a better job. We need to change the STG -\> Cmm code generator to attach register liveness information at function exit points (e.g calls, jumps, returns).



e.g This [
bug (\#4308)](http://hackage.haskell.org/trac/ghc/ticket/4308) is as a result of this problem.

