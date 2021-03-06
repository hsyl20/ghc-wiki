
**For type applications within function application expressions (e.g. f \@Int x), see [TypeApplication](type-application)**


# Syntax for explicit kinds, new story



When working with kind polymorphism and promotion we often want to 
specify kinds independently of types. In fact, we might not even be
interested in the types, only in kinds. This is fine in FC-pro, but
we need syntax to express it in source Haskell. We propose to allow
choosing at the definition site to either:


1. give explicit kind parameters, or
1. infer kind parameters


If you choose (a) you must always give explicit kind arguments.
If you choose (b) you cannot ever give explicit kind arguments. 



Relevant tickets (at least)


- [\#5296](http://gitlabghc.nibbler/ghc/ghc/issues/5296)
- [\#4466](http://gitlabghc.nibbler/ghc/ghc/issues/4466)


See also [ImpredicativePolymorphism](impredicative-polymorphism).


## Examples



(b) above is the current behaviour; you can only specify kinds by
annotating types:


```wiki
type family App (f :: k1 -> k2) (a :: k1) :: k2
```


We typically want to use (a) when we are not interested in types, only kinds.
For instance, consider the following type family, which is a variant of the
[
type-level literals module](https://github.com/ghc/packages-base/blob/master/GHC/TypeLits.hs):


```wiki
type family SingRep (k :: ☐)

type instance SingRep Nat    = Integer
type instance SingRep Symbol = String
```


Note that `Nat` and `Symbol` above are **kinds**.



**Question: ** do we need to use `'Nat` instead of `Nat` above for the renamer
to know we want the kind and not the type `Nat`?



Explicit kind arguments can also be used in type classes, and function type
signatures:


```wiki
class SingE (k :: ☐) where
  fromSing :: Sing (k :: ☐) -> SingRep (k :: ☐)

instance SingE Nat where
  fromSing (SNat n) = n

instance SingE Symbol where
  fromSing (SSym s) = s
```


**Question: ** do we want/need explicit kind arguments at the value level?


## How to distinguish kind variables



In the examples above we have used the `k :: ☐` notation to indicate that `k`
is a kind variable. We do not want to require unicode usage for this, so we
propose either:


1. `k :: BOX`, or
1. `@k`


(1) makes `BOX` a reserved name. (2) does not require reserving any names, but
will not work on value-level patterns, since it can be confused with an 
at-pattern. (However, see below for a possible solution to this problem.) 


# Syntax for explicit type and kind application, old story



We now describe a more general proposal that allows explicit type and kind application, but is better
suited for specifying arguments at the call site. This makes it easier to work with impredicative
polymorphism, but not with kind families, for instance.



We propose a replacement and generalisation of [lexically-scoped type variables](http://www.haskell.org/ghc/docs/latest/html/users_guide/other-type-extensions.html#scoped-type-variables) (and pattern signatures) that is
more clear and direct by allowing explicit type (and kind) application.
We propose the concrete syntax `@ tyvar`, like in the following example:


```wiki
case x of
  (C @a y z) -> ....
```


On the right-hand side we would have the type variable `a` in scope for use on 
any type signatures.



Note how the use of the symbol `@` is (in this case) unproblematic; we can
use the fact that constructors always start with an uppercase letter to distinguish
whether the `@` refers to an "as pattern" or to a type application:


```wiki
case x of
  p@(C @a y z) -> ....
```


Unfortunately this is not always the case; see below.



Note that this proposal would not allow pattern matching on specific types:
the only thing that we can match on are type or kind variables. However, it
does allow for specifying what type to apply:


```wiki
id @Int 2
```


The idea is to provide access to the explicit types in the core language
(system [ FC-pro](http://dreixel.net/research/pdf/ghp.pdf))
directly from the source language syntax.


## How many arguments, and their order



When we have multiple variables we can pattern match on as many as we need,
and also use underscores:


```wiki
f (C @_ @b x ) = ...
```


If the user gave a type signature for the binding, it is very easy to see
which type patterns relate to which variables in the signature. In the absence
of a signature, though, there are two possible choices:


- Reject matching on type variables altogether.

- Take the inferred signature, look at the introduced variables syntactically
  from left to right, and use that order. This approach does not require tracking
  which bindings were given type signatures or not.


A problem with taking the inferred signature is that it is tied to
many assumptions, including that of principal types.
\[Dimitrios: Can you expand on this?\]


## Parsing matters


### Ambiguity



Consider a problematic example:


```wiki
f :: Int -> forall a. a
f n @a = ....
```


In this case it is really ambiguous what the pattern means. For these
cases we suggest the following workaround:


```wiki
f :: Int -> forall a. a
(f n) @a = ....
```


This approach should work in general, and hopefully only few programs will
actually need to use it.


### Other Syntax Proposals



Here are some other examples of syntax that could be used for explicit type application:


```wiki
f :: forall a b c. a -> b -> c -> (a, b, c)

f @Int @Bool @Char 3 True 'a'      {- Similar to above -}
f {Int} {Bool} {Char} ...          {- Agda; potential record conflict -}
@f Int Bool Char ...               {- Coq -}
#f Int Bool Char ...
```

### Syntax for promoted datatypes



With `-XPolyKinds` on, we can also match/apply kind arguments. This introduces the
need to disambiguate between a datatype and the promoted kind it introduces.
Consider the example:


```wiki
data X = X

f :: forall (a : k). ....
... = ... f @'Nat @Nat ...
```


Since now it is not clear from the context anymore if we are expecting a kind
or a type (since we want to use `@` both for kind and type application), we need to be
able to disambiguate between datatypes and their corresponding promoted kinds.
At the moment this ambiguity does not arise, so we do not allow prefixing
kinds with `'`, but it seems natural to lift this restriction, and use the
same notation as for promoted data constructors.


## More examples


### Impredicative instantiation



This extension also allows for clear impredicative instantiation. For instance,
the application of the list constructor `(:) @(forall a. a -> a)` means
the constructor of type
`(forall a. a -> a) -> [forall a. a -> a] -> [forall a. a -> a]`.



NB: As of GHC 8.0 this is not actually supported without `-XImpredicativeTypes`.


### Type/kind instantiation in classes



With the new kind-polymorphic `Typeable` class, we can recover the old
kind-specific classes by writing, for example:


```wiki
type Typeable1 = Typeable @(* -> *)
```

### Further Information



For more information on explicit type application, see [\#4466](http://gitlabghc.nibbler/ghc/ghc/issues/4466).


