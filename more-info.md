
javran is working on extending `:info` to support more general cases (as requested in [\#9394](http://gitlabghc.nibbler/ghc/ghc/issues/9394)), here is the detail of my plan.


# Motiviation



Current implementation of `:info <things>` of GHCi simply splits whatever in `<things>` and prints information for each of them in order.
The motivation is to extend `:info` to parse info query better and support searching data / type family instances.


# Design of New `:info` Syntax


- Keep the old behavior that if you input multiple queries, all of them are executed in sequence. For example `:info Int Char` has the same behavior as before, `:info Int (Foo Bar)` first searches things regarding `Int` and then things related to type application `Foo Bar`.

- When `(` is present, expect a type and a closing `)` (type can contain holes to serve as wildcards), after parsing this type, we search it against instances related (not sure about the detail, maybe some type checking facilities will do)

- There are some special cases where `(` can also begin a regular query: `()`, `(,)`, `(,,,)`, they are treated in the old way.

# Dealing With More General Queries



Most of the following are copied from s9gf4ult comments on [\#9394](http://gitlabghc.nibbler/ghc/ghc/issues/9394), as I myself am still new to some of those concepts.
Hopefully I'll make myself comfortable with them when getting into details


- Finite typeclass application: For finite typeclass application we can print detailed information about specific instance, for example `:info (Eq Int)` must print assigned type/data families, module where instance is defined, maybe something else.

- Finite data type: as application type parameters to ordinary data type with parameters. Here we can print the same as for other finite type like Int

- Finite data family application: print information like for any other finite type, or, if type is not finite

- Finite type family application: print type assigned to this type family.

# Roadmap



I'll like to split the plan into multiple steps, a rough plan is to first aim for querying data/type family instances while getting to detailed plan about handling more general cases.


