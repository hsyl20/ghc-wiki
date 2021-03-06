# Idiom: phase ordering



NB. you need to understand this section if either (a) you are modifying parts of the build system that include automatically-generated `Makefile` code, or (b) you need to understand why we have a top-level `Makefile` that recursively invokes **make**.



The main hitch with non-recursive **make** arises when parts of the build
system are automatically-generated.  The automatically-generated parts
of our build system fall into two main categories:


- Dependencies: we use `ghc -M` to generate make-dependencies for 
  Haskell source files, and similarly `gcc -M` to do the same for
  C files.  The dependencies are normally generated in the
  [distdir](building/architecture/idiom/distdir) directory
  into a `.depend` file, which is included as normal.

- Makefile bindings generated from `%.cabal` package descriptions.  The
  bindings are stored in `package-data.mk` files in the
  [distdir](building/architecture/idiom/distdir) directory, see
  [Idiom: interaction with Cabal](building/architecture/idiom/cabal).


Now, we also want to be able to use `make` to build these files, since
they have complex dependencies themselves.  For example, in order to build
`package-data.mk` we need to first build `ghc-cabal` etc.; similarly,
a `.depend` file needs to be re-generated if any of the source files have changed.



GNU **make** has a clever strategy for handling this kind of scenario.  It
first reads all the included Makefiles, and then tries to build each
one if it is out-of-date, using the rules in the Makefiles themselves.
When it has brought all the included Makefiles up-to-date, it restarts itself
to read the newly-generated Makefiles.



This works fine, unless there are dependencies *between* the
Makefiles.  For example in the GHC build, the `.depend` file for a
package cannot be generated until `package-data.mk` has been generated
and **make** has been restarted to read in its contents, because it is the
`package-data.mk` file that tells us which modules are in the package.
But **make** always makes **all** the included `Makefiles` before restarting - it
doesn't know how to restart itself earlier when there is a dependency
between included `Makefiles`.



Consider the following Makefile (called `fail.mk`):


```wiki
all :

include inc1.mk

inc1.mk : fail.mk
	echo "X = C" >$@

include inc2.mk

inc2.mk : inc1.mk
	echo "Y = $(X)" >$@
```


Now try it:


```wiki
$ make -f fail.mk
fail.mk:3: inc1.mk: No such file or directory
fail.mk:8: inc2.mk: No such file or directory
echo "X = C" >inc1.mk
echo "Y = " >inc2.mk
make: Nothing to be done for `all'.
```


**make** built both `inc1.mk` and `inc2.mk` without restarting itself
between the two (even though we added a dependency on `inc1.mk` from
`inc2.mk`).



The solution we adopt in the GHC build system is as follows.  We have
two Makefiles, the first a wrapper around the second.


```wiki
# top-level Makefile
% :
        $(MAKE) -f inc.mk PHASE=0 just-makefiles
        $(MAKE) -f inc.mk $<
```

```wiki
# inc.mk

include inc1.mk

ifeq "$(PHASE)" "0"

inc1.mk : inc.mk
	echo "X = C" >$@

else

include inc2.mk

inc2.mk : inc1.mk
	echo "Y = $(X)" >$@

endif

just-makefiles:
        @: # do nothing

clean :
	rm -f inc1.mk inc2.mk
```


Each time **make** is invoked, we recursively invoke **make** in several
*phases*:


- **Phase 0**: invoke `inc.mk` with `PHASE=0`.  This brings `inc1.mk` 
  up-to-date (and *only* `inc1.mk`).  

- **Final phase**: invoke `inc.mk` again (with `PHASE` unset).  Now we can be sure 
  that `inc1.mk` is up-to-date and proceed to generate `inc2.mk`.  
  If this changes `inc2.mk`, then **make** automatically re-invokes itself,
  repeating the final phase.


We could instead have abandoned **make**'s automatic re-invocation mechanism altogether,
and used three explicit phases (0, 1, and final), but in practice it's very convenient to use the automatic
re-invocation when there are no problematic dependencies.



Note that the `inc1.mk` rule is *only* enabled in phase 0, so that if we accidentally call `inc.mk` without first performing phase 0, we will either get a failure (if `inc1.mk` doesn't exist), or otherwise **make** will not update `inc1.mk` if it is out-of-date.



In the case of the GHC build system, the two Makefiles are called `Makefile` (the wrapper) and `ghc.mk`.



This approach is not at all pretty, and
re-invoking **make** every time is slow, but we don't know of a better
workaround for this problem.


## GHC's phases



In the GHC build system, we have 3 phases. Each phase is in two halves: some things are built while make is rebuilding makefiles (the "includes" half), and some things are built as they are the target we ask make to build (the "builds" half).


- Phase 0

  - Includes: `package-data.mk` files for things built by the bootstrapping compiler.
  - Builds: the dependency files for `hsc2hs` and `genprimopcode`. We need to do this now, as `hsc2hs` needs to be buildable in phase 1's includes (so that we can make the `hpc` library's `.hs` source files, which in turn are necessary for making its dependency files), and `genprimopcode` needs to be buildable in phase 1's includes (so that we can make the `primop-*.hs-incl` files, which are sources for the stage1 compiler library, and thus necessary for making its dependency files).
- Phase 1

  - Includes: dependency files for things built by the bootstrapping compiler.
  - Builds: `package-data.mk` files for everything else. Note that this requires configuring the packages, which means telling cabal which ghc to use, and thus the stage1 compiler gets built during this phase.
- Phase "final"

  - Includes: dependency files for everything else.
  - Builds: Everything else.


See the comments in the [top-level ghc.mk](/trac/ghc/browser/ghc.mk)[](/trac/ghc/export/HEAD/ghc/ghc.mk) for details
(search for 'Approximate build order' and 'Numbered phase targets').


