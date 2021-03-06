# Type-Indexed Records



Proposal for new Haskell record system. Record selection is simple operator. Keys are arbitrary types. Scope is controlled as scope of key types.


# Basics



Type classes for types with member at 'k'


```wiki
class Has k v r where (.) :: r -> k -> v;
```


which means that r has member of type v with key type k, and for types with mutable member at 'k'


```wiki
class (Has k u r, Has k v s) => Quasifunctor k u v r s where qfmap :: k -> (u -> v) -> r -> s;
```


which means that r and s have members of types u and v, in turn, both with selector k; thus, one can mutate the member at 'k' with an arbitrary function of type u -\> v, and the overall function is of type r -\> s; i.e. one can lift a function of type u -\> v to a function of type r -\> s.



A record type is of the form


```wiki
type R a b c ... = { X ::. a, Y ::. b, Z ::. c, ... };
```


which automatically generates


```wiki
instance Has X a (R a b c ...);
instance Has Y b (R a b c ...);
instance Has Z c (R a b c ...);
...
instance Quasifunctor X a a' (R a b c ...) (R a' b c ...);
instance Quasifunctor Y b b' (R a b c ...) (R a b' c ...);
instance Quasifunctor Z c c' (R a b c ...) (R a b c' ...);
...
```

## Record selection and mutation



Let


```wiki
type R a b c = { X ::. a, Y ::. b, Z ::. c, ... };

-- keys
data X = X;
data Y = Y;
data Z = Z;
```


Then `r.X :: a` is the member of `r` at `X`, and `qfmap X f r` is `r` mutated by `f` at `X`; thus also for other keys `Y`, `Z`, ....
We might define some sugar for `qfmap`.



We can define


```wiki
x = X;
y = Y;
z = Z;
```


to allow `r.x`, `r.y`, `r.z`, ....


