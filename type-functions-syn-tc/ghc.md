## Special forms of type equations






We impose some syntactic restrictions on type instances and normalise equations arising during type checking to ensure that type checking remains (a) deterministic, (b) a simple rewrite model is sufficient to reduce type function applications, and (c) it hopefully remains decidable.


### Type variables



We have three kinds of type variables:


- *Schema variables*: These are type variables in the left-hand side of type equations that maybe instantiated during applying a type equation during type rewriting.  For example, in `type instance F [a] = a`, `a` is a schema variable.
- *Rigid variables*: These are variables that may not be instantiated (they represent variables in signatures and existentials during matching).
- *Wobbly variables*: Variables that may be instantiated during unification.

### Normal type equations



*Normal type equations* `s = t` obey the following constraints:


- *Constructor based*: The left-hand side `s` must have for form `F s1 .. sn` where `F` is a type family and the `si` are formed from data type constructors, schema variables, and rigid variables only (i.e., they may not contain type family constructors or wobbly variables).
- *Non-overlapping*: For any other axiom or local assumption `s' = t'`, there may not be any substitution *theta*, such that (*theta* `s`) equals (*theta* `s'`).
- *Left linear*: No schema variable appears more than once in `s`.
- *Decreasing*: The number of data type constructor and variables symbols occurring in the arguments of a type family occuring in `t` must be strictly less than the number of data type constructor and variable symbols in `s`.
- *Rank 0*: No foralls in `s` or `t`.


Examples of normal type equations:


```wiki
data C
type instance Id a = a
type instance F [a] = a
type instance F (C (C a)) = F (C a)
type instance F (C (C a)) = F (C (Id a))
type instance F (C (C a)) = C (F (C a))
type instance F (C (C a)) = (F (C a), F (C a))
```


Examples of type equations that are *not* normal:


```wiki
type instance F [a] = F (Maybe a)            -- Not decreasing
type instance G a a = a                      -- Not left linear
type instance F (G a) = a                    -- Not constructor-based
type instance F (C (C a)) = F (C (Id (C a))) -- Not decreasing
```


Note that `forall a. (G a a = a) => a -> a` is fine, as `a` is a rigid variables, not a schema variable.



We require that all type family instances are normal.  Moreover, all equalities arising as local assumptions need to be such that they can be normalised (see below).  NB: With `-fundecidable-indexed-types`, we can drop left linearity and decreasingness.


### Semi-normal type equations



If an equation `s = t` does not contain any schema variables and is normal, except that it's left-hand side `F s1 .. sn` contains one or more type family constructors in the `si`, we call it *semi-normal*.


### Normalisation of equalities



Normalisation of an equality `s = t` of arbitrary rank-0 type terms `s` and `t` (not containing schema variables) leads to a (possibly empty) set of normal equations, or to a type error.  We proceed as follows:


1. Reduce `s` and `t` to NF, giving us `s'` and `t'`.
1. If `s'` and `t'` are identical, we succeed (with no new rule).
1. If `s'` or `t'` is a rigid variable, we fail.  (Reason: cannot instantiate rigid variables.)
1. If `s'` or `t'` is a wobbly type variables, instantiate it with the other type (after occurs check).
1. If `s'` = `C s1 .. sn` and `t'` = `C t1 .. tn`, then yield the union of the equations obtained by normalising all `ti = si`.
1. If `s'` = `C1 ...` and `t' = C2 ...`, where `C1` and `C2` are different data type constructors, we fail.  (Reason: unfication failure.)
1. Now, at least one of `s'` and `t'` has the form `F r1 .. rn`, where F is a type family:

  - If `s' = t'` is normal, yield it.
  - If `t' = s'` is normal, yield it.
  - If `s' = t'` is semi-normal, yield it.
  - If `t' = s'` is semi-normal, yield it.
  - Otherwise, fail.  (Reason: a wobbly type variable, lack of left linearity, or non-decreasingness prevents us from obtaining a normal equation.  If it is a wobbly type variable, the user can help by adding a type annotation; otherwise, we cannot handle the program without (maybe) losing decidability.)


Rejection of local assumptions that after normalisation are either not left linear or not decreasing may lead to incompleteness.  However, this should only happen for programs that are invalid or combine GADTs and type functions in ellaborate ways.


## Maintaining type equations



The set of *given* equalities (i.e., those that we use as rewrite rules to normalise type terms) comprises two subsets:


- *Axioms*: The equations derived from type family instances.  They are the only equations that may contain schema variables, and they are normal for well-formed programs.
- *Local assumptions:* The equations arising from equalities in signatures and from GADT pattern matching after normalisation.


The set of axioms stays the same throughout type checking a module, whereas the set of local assumptions grows while descending into expressions and shrinks when ascending out of these expressions again.  We have two different sorts of local assumptions:


- *Normal assumptions*: These are not altered anymore once they have been added to the set of local assumptions, until the moment when they are removed again.
- *Semi-normal assumptions:* These are only added tentatively.  They are reconsidered whenever a new rule is added to the local assumptions, because a new rule may lead to further normalisation of semi-normal assumptions.  If a semi-normal assumption is further normalised, the original assumption is removed and the further normalised one added (which can again trigger subsequent normalisation).  NB: Strictly speaking, we can leave the original (semi-normal) equation in the set together with its further normalised version.

## Termination



To guarantee termination of type inference, we need to require that all local assumptions are decreasing.  The good thing about decreasingness is that it is a cheap syntactic check.  The bad thing is that it is overly conservative (so we reject some good programs that could never lead to non-termination).  An alternative and more precise check for whether a new equation can lead to non-termination is to (1) convert that equation and all local assumptions and axioms it depends on (directly or indirectly) to CHR form, (2) apply CHR rules until the final store is reached, and (3) check that store for consistency.  This is more precise, but also less efficient.


