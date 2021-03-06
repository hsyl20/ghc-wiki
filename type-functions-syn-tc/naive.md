## A first (naive) attempt



To solve  `(C implies t1=t2)` with respect to Ax


1. We interpret Ax /\\ C as a rewrite system (from left to right).
1. We exhaustively apply rewrite rules on t1 and t2, written t1 --\>\* t1' and t2 --\>\* t2' and check that t1' and t2' are syntactically equivalent.


Immediately, we find a problem with this solving strategy.
Consider our running example.



Rewrite rules


```wiki
(forall a. S [a] = [S a]) /\      (R1) -- axioms
(T Int = Int)                     (R2)

 /\

(T [Int] = S [Int]) /\            (R3) -- local assumptions
(T Int = S Int)                   (R4)
```


applied to `(T [Int] = [Int])`



yields


```wiki
T [Int] -->* [S Int]       (R3,R1)

[Int] -->* [Int]
```


Hence, our (naive) solver fails, but
clearly the (local) property  (T \[Int\] = \[Int\])
holds.



The trouble here is that


- the axiom system Ax is confluent, but
- if we include the local assumptions C, the combined system Ax /\\ C is non-confluent (interpreted as a rewrite system)


Possible solutions:



Enforce syntactic conditions such that Ax /\\ C is confluent.
It's pretty straightforward to enforce that Ax and
constraints appearing in type annotations and data types
are confluent. The tricky point is that if we combine
these constraints they may become non-confluent.
For example, imagine


```wiki
Ax : T Int = Int

   a= T Int      -- from f :: a=T Int => ...

      implies 

        (a = S Int -- from a GADT pattern

            implies ...)
```


The point is that only during type checking we may
encounter that Ax /\\ C is non-confluent!
So, we clearly need a better type checking method.


