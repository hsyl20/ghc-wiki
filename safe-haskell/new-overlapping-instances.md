# Safe Haskell & Overlapping Instances



Overlapping Instances cause problems for Safe Haskell. Why? Because we
don't want code compiled in the `-XSafe` language to be able to alter
the behavior of other code in a indirect manner through instance
selection.



That is, overlapping instances has the effect of changing the behavior
of existing code simply through importing a new module and bringing
new instances into scope. As a primary use case of Safe Haskell is
building new libraries for enforcing security on untrusted code, this
is problematic.


## Safe Haskell and Security



In the security setting, untrusted code is compiled with `-XSafe` to
ensure the types can be trusted and the code can't violate any
invariants enforced through the type system. The TCB of the security
enforcement library, will most likely contain code written using
`-XUnsafe`. This is to firstly allow access to all parts of the
greater GHC Haskell language and also to disallow untrusted code from
importing the module and gaining access to constructors that may allow
it to violate invariants you want enforced. This works since code
compiled with `-XSafe` can only import other `-XSafe` modules or
`-XTrustworhty` modules.



While `-XTrustworthy` modules may themselves import `-XUnsafe`
modules, the user compiling the code must trust the package that any
`-XTrustworthy` module resides in to indicate that they trust the
author of the `-XTrustworthy` module to have only exposed a safe
interface despite the unsafe internals.


## GHC 7.10 & Earlier



This is how overlapping instances were handled in GHC 7.10 and earlier.


### Instance Selection



In the security use-case, if we have code involving overlapping
instances, we don't want to compile any cases where instances from
`-XSafe` modules overlap instances from `-XUnsafe` modules. Why?
Because now the behavior desired by the security system author may
have been overridden by an untrusted, potentially malicious author.



To deal with this, we apply a simple rule. When resolving instance
selection during compilation, if the most specific instances comes
from a module M that was compiled with `-XSafe`, then all other
instances we considered (i.e., all the less specific, overlapped
instances) must also be from M. You could imagine relaxing this a
little and just enforcing that all overlapped instances are also from
`-XSafe` modules, but we prefer to isolate `-XSafe` modules instead
and have a stronger guarantee.



This essentially is a same-origin-policy, `-XSafe` modules can use
overlapping instances, but can only overlap themselves and no one
else.


### Instances and Safe-Inference



How do we infer safety given the above policy? We take a conservative
approach (by necessity) and consider a module that enables the
`-XOverlappingInstances` flag as `-XUnsafe`.



This is since compiling a module with `-XSafe` means those instances
can no longer overlap any instances outside the module. As overlapping
instances is only resolved at the call-site and not before, this
largely affects the consumers of a module compiled with `-XSafe` and
how it affects them changes when their own code changes. (I.e.,
removing an instance from an `-XUnsafe` module may now make a
call-site of overlapping instances compile since perhaps all possible
instances except the one removed came from the same `-XSafe` module).



As we can't change the semantics of Haskell when inferring safety, we
need to be conservative.


#### Bug in Safe vs Safe-Inferred



Interestingly, this isn't exactly true. Consider the following module:


```wiki
{-# LANGUAGE FlexibleInstances #-}
module S where

class C a where
  f :: a -> String

instance C [Int] where
  f _ = "[Int]"
```


This will compile fine as-is and doesn't require a
`-XOverlappingInstances` flag as we only consider when instances
overlap at call-sites, not at declaration. It will be inferred as
safe. Now consider this module:


```wiki
{-# LANGUAGE Unsafe #-}
{-# LANGUAGE OverlappingInstances #-}
{-# LANGUAGE FlexibleInstances #-}
module T where

import safe S

instance C [a] where
  f _ = "[a]"

test :: String
test = f ([1,2,3,4] :: [Int])
```


The above should actually fail to compile since we have the instances
`C [Int]` from the `-XSafe` module S overlapping as the most specific
instance the other instance `C [a]` from module T. This is in
violation of our single-origin-policy.



Right now though, the above actually compiles fine but \*this is a
bug\*. Compiling module S with `-XSafe` has the right affect of causing
the compilation of module T to then subsequently fail. So we have a
discrepancy between a safe-inferred module and a `-XSafe` module,
which there should not be.



It does raise a question of if this bug should be fixed. Right now
we've designed Safe Haskell to be completely opt-in, even with
safe-inference. Fixing this of course changes this, causing
safe-inference to alter the compilation success of some cases. How
common it is to have overlapping declarations without
`-XOverlappingInstances` specified needs to be tested.


## GHC 7.12 & Later



We have significantly changed how overlapping instances work in 
GHC 7.12 and later. This is both to improve the design, and also to
make use of the new overlapping instance pragmas added in GHC 7.10.


### Overlapping Instance Pragmas



GHC 7.10 added in instance specific pragmas to control overlapping
of instances. They consist of:


- `OVERLAPPABLE` -- Specifies that the instance author allows this
  instance to be overlapped by others.
- `OVERLAPPING` -- Specifies that the instance author is expecting
  this instance will overlap others.
- `OVERLAPS` -- Implies both `OVERLAPPABLE` and `OVERLAPPING`. This is
  equivalent to the old `-XOverlappingInstances` behavior.

### Problems with GHC 7.10 Approach


#### Can't detect declaration of overlapping instances



We previously try to detect when a module declared overlapping
instances and mark it unsafe when it did (at least for inference).
This doesn't work. Overlapping instances are a global property.
We only know when an overlap occurs at a particular call site for
a type-class method.



For example, it is perfectly reasonable to write two modules
as follows:


```wiki
module A where

  instance C [a] where { op = ... }

module B where

   instance C [Int] where { op = ... }
```


Neither module in isolation declares overlapping instances. It's
only when we import both modules and call `op` that we can detect
the overlap.



Due to the new instance specific pragmas in GHC 7.10, we could 
simply declare all modules that declare `OVERLAPPABLE` instances
to be unsafe. But this would be heavily over-approximate.



A closer approximation would be to declare all orphan instances
that are marked `OVERLAPPABLE` as unsafe. Since, if at a
type-class method call site, if the instance selected is not
an orphan instance, then I must either depend on the type
declared in the same module as the instance, or I must depend
on the type-class declared in the same module. In both situations
the dependency is explicit, so safe.



However, MPTC make this tricky. So we decided to detect unsafe
overlaps at call-sites, not at declaration.


#### Safe Inference



GHC 7.10 had a different rule for `-XSafe` modules and
safe-inferred modules with respect to overlapping instances.
This lack of symmetry is a huge problem. See the bugs it
causes for example, list earlier.


### GHC 7.12 Approach



We detect unsafe overlaps at call-sites and ensure symmetry
between `-XSafe` and safe-inference.



The rule to determine if a particular call-site of a type-class
method is **unsafe** is:


- Most specific instance, **Ix**', defined in an `-XSafe`
  compiled module.
- **Ix** is an orphan instance or a multi-parameter-type-class.
- At least one overlapped instance, **Iy**, is both:

  - From a different module than **Ix**
  - **Iy** is not marked `OVERLAPPABLE`.

### Where this check applies



This check is enforced in `-XSafe` and `-XTrustworthy` modules.
We also use it to infer safety. `-XUnsafe` modules don't have the
check enforced.



A nice extension may be to tie this check into whether a module was
imported as a safe import or not. This wouldn't change `-XSafe`
modules, but would allow selective application of the rule to
`-XTrustworthy` and `-XUnsafe` modules. So far we haven't done this
as it isn't clear where an instance is imported from, since they
can be brought into scope recursively by multiple imports. It also
complicates safe imports and isn't clear if the gain is worth it.



One way forward would be to implement this extension, and when an
import is imported through multiple paths, to take the conservative
join of the two.


