## Implementation plan for `-XStaticPointers`



**Read the [StaticPointers](static-pointers) proposal first.**



The `-XStaticPointers` language extension as currently envisioned is very lightweight, adding just one new syntactic form to expressions with its attendant typing rule. However, in order to implement the language extension with a smaller trusted code base, a few changes need to happen in unrelated parts of the compiler and base libraries. In particular, in order to avoid unsafe type casts in the implementation of [distributed-closure](distributed-closures), we will need the [typed Typeable](typeable) proposal. Further, in order to enable dynamic checks that static pointers are always decoded against the right type (recall that `StaticPointer a` has a phantom type parameter `a` which must match the type of the value referred to), GHC must maintain a Static Pointer Table (SPT) including one entry for each (unique) expression `e` appearing in the body of a `static e` form. This table is looked up at runtime, during decoding of static pointers.


## Interim plan for 7.10



However, there is a short-term workable plan for GHC 7.10, that does not require redefining the `Data.Typeable` class at the last minute of the release engineering cycle. We observe that:


- The typed `Typeable` proposal makes it possible to avoid unsafe type casts in the presence of existentials (as does appear in the implementation of distributed-closure), but is otherwise not required. In other words, that proposal helps reduce the size of trusted base, but does not otherwise enable any particular feature.

- The SPT is required to make the following code raise an exception gracefully, rather than segfault: `decode (encode (ptr :: StaticPtr a)) :: StaticPtr b`. The *metadata* stored in the SPT, in particular the type of each value that is stored in it, is only useful for extra dynamic checks (which some people will want to opt out of anyways for performance), but not required for implementing the functionality of the language extension (decoding can still be done without it, albeit less safely).

- Users normally do not encode / decode static pointers directly, nor even do they use most of distributed-closure directly. Instead, a higher-level application-specific framework such as distributed-process is used. In the particular case of distributed-process, the dynamic type checks that the SPT enables should normally be redundant with the checks that distributed-process itself performs. To be clear, use of the SPT means we *don't need to trust the implementation of distributed-process to avoid segfaults* (see last section below), but not making use of the SPT for now does not mean the user is at risk of segfaulting by mistake.


The proposal is therefore to:


- Implement the extension in two phases: the full version with full dynamic type checks on top of a revised `Data.Typeable` in GHC 7.12, but an interim version in GHC 7.10 **with a similar API** but missing the dynamic type checks enabled by the SPT.

- In GHC 7.10, *implement only a shim SPT*, that looks just like the real thing, but does not include `tTypeRep`s or `TypeRep`s for any value. The reason we still need an SPT in GHC 7.10 is because there needs to be something that references static expressions throughout the lifetime of the program. The reason is that any such expression may at any time be referenced by a new `StaticPtr` produced from an incoming message. In other words, in a distributed system, other nodes may refer to values that are otherwise not live on the local node. The SPT must itself be protected against garbage collection, e.g. through the use of a `StablePtr`.

### The API of `GHC.StaticPtr` (interim version for 7.10)


```wiki
module GHC.StaticPtr
  ( StaticPtr
  , StaticKey
  , StaticPtrInfo(..)
  , deRefStaticPtr
  , staticPtrKey
  , unsafeLookupStaticPtr
  , staticPtrInfo
  ) where

-- | A key for `StaticPtrs` that can be serialized and used with 'unsafeLookupStaticPtr'.
type StaticKey = Fingerprint

-- | Miscelaneous information available for debugging purposes.
data StaticPtrInfo = StaticPtrInfo { pkgId, moduleName :: String, ... }

deRefStaticPtr        :: StaticPtr a -> a
staticPtrKey          :: StaticPtr a -> StaticKey
staticPtrInfo         :: StaticPtr a -> StaticPtrInfo
unsafeLookupStaticPtr :: StaticKey   -> Maybe (StaticPtr a)

data StaticPtr
```


**Remarks:**


- `deRefStaticPtr` so named to make its name consistent with `deRefStablePtr` in `GHC.StablePtr`.
- This module will be added to `base`, as for other primitives exposed by the compiler. This means we cannot depend on `bytestring` or any other package except `ghc-prim`.
- As such, we should leave it up to user libraries how they wish to encode `StaticPtr`, using whatever target type they wish (e.g. `ByteString`). The solution is to provide a serializable `StaticKey` for each `StaticPtr`.
- The definition of `StaticKey` must be part of the public API to make this work: otherwise there would be no way for the user to define her own encoder.
- Encoders and decoders are normally overloaded functions of the 'binary' package, another package not included in base. It should be up to the application or framework to define these, in user libraries.
- The above is the full extent of what needs to go into the base libraries and/or the compiler. Everything else, including distributed-closure, the definition of `Closure`, the `Serializable` type class, etc, can live in a separate package and need not be tied to any particular version of GHC.

---


## Implementation notes for interim version



** `StaticPtr` is defined as follows:
**


```wiki
data StaticPtr a = StaticPtr !Fingerprint StaticPtrInfo a
  deriving Typeable
```

- The compiler passes work like this

  - **Lexical analysis**: `static` is a keyword when `-XStaticPointers` is on

  - **Parsing**: new constructor `HsStatic` in `HsExpr`, to represent `(static e)`.

  - **Renaming**: in `(static e)`, check that all free variables of `e` are bound at top level. And check that static does not occur in splices (but can occur in quotations).

  - **Typechecking**. In the type checker, we add `Typeable a` to the set of constraints, and we treat the body of the static form as if it were a top level definition.

  - **Desugaring**: desugar `(static e)` to `sptEntry:0`, and create a new top-level binding

    ```wiki
    sptEntry:0 :: StaticPtr (type of e)
    sptEntry:0 = StaticPtr (fingerprintString "packageId:module.sptEntry:0")  (StaticPtrInfo ...) e
    ```

    where `sptEntry:0` is a fresh name.

  - **Code generation**.  All such `sptEntry:*` definitions are considered SPT entries. Before `main` is invoked, modules are  initialized by inserting all their SPT entries into a global SPT which lives in the RTS. This initialization is implemented via [
    constructor functions](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html) in the same way that module initialization is implemented for [
    HPC](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Hpc) (Coverage.hpcInitCode).

- A `StablePtr` for each entry is created to avoid it being garbage collected (can we register the SPT as a source of roots with a single call?).
- The SPT is a hash table mapping `Fingerprint`s to the closures of the SPT entries.


**Remark:** do we need `sptEntry:0` at all? It looks like we could generate the right code with all the entries, without the extra indirection.
**Tentative answer:** at module initialization time, we need pointers to the SPT entries that we can insert into the SPT. Making SPT entries top-level definitions is a convenient mechanism to get such pointers. To avoid producing top-level definitions, we would need an alternative mechanism.


### Appendix: notes about distributed-process



Messages sent across the wire by distributed-process fall in one of three categories:


- control messages, which always have type `NCMsg`, a monomorphic type;
- type tagged messages sent using `send`, received using `expect` or `receiveWait`. These messages always include a header containing the fingerprint of the `TypeRep` for the value (needed to that `expect` and friends can dispatch on messages of the right type on the receiving end);
- channel messages, which don't include a type tag, since the type is implied by the channel on which the message is delivered.


In each case, without an SPT, one must trust that distributed-process is deserializing at the right type when deserializing closures. One does not, however, need to trust the user in order to rule out the possibility of segfaults, so long as she is *only* a client of said library (i.e. does not use any side channel to send closures across the network).


