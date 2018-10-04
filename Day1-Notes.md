# Day 1

Type = a set of elements
Isomorphic type : same number of elements as another type, it's then possible to map from one to another back and forth

A function that takes Unit as param, can always be called (just provide it Unit)

## Product / Sum types

Product type C = A X B
A and B are the "Terms" of the Product
C is a 2-way product
C "set" size is A size times B size

type 1 is Unit
type 0 is Nothing

A X 1 ~= A (its isomorphic)
1 X A ~= A

2-way product is enough to encode all n-way products


Sum types : 2 ways to represent
 - Either
 - sealed trait

A + 0 ~= A
0 + A ~= A

```
def to1[A](t: (A, Unit)): A = t._1

def from1[A](a: A): (A, Unit) = (a, ())

val x1 = to1(from1(42))

def to2[A](t: Either[A, Nothing]): A = t match {
  case Left(a) => a
  case Right(n) => n: A
}

def from2[A](a: A): Either[A, Nothing] = Left(a)

val x2 = to2(from2(42))
```

Inheritance is used only to build sum types, sealed traits never have any method
in FP, never use inheritance for anything else


## The Id problem (illegal states)

We have a set or Realms (small finite number)
We use `Int` as an id for those realms
Pb: Int is a very large type, much larger than the set of realms. There are many "Illegal states" than correspond to no realm, that would never be used.
What do: create "new type"

```
class RealmId private (value: Int)
object RealmId {
  def apply(id: Int): Option[RealmId] = ???
}
```

```
case class GameMap(realms: List[Realm], paths: List[(RealmId, RealmId)])

class RealmId private (value: Int)
object RealmId { def apply(id: Int): Option[RealmId] = ??? }
case class Realm(id: RealmId, realmType: RealmType, description: String, inv: List[Item], chars: List[Character])

sealed trait RealmType
case object Plains extends RealmType
case object Highlands extends RealmType
case object Caves extends RealmType
case object Indoors extends RealmType
case object Underwater extends RealmType

sealed trait Item
case class Character(inv: List[Item], charType: CharType) sealed trait CharType
case class Player(inv: List[Item]) extends CharType
case class NonPlayerCharacter(npcType: NPCType, inv: List[Item]) extends CharType

sealed trait NPCType
case object Ogre extends NPCType
case object Troll extends NPCType
case object Wizard extends NPCType
case object John extends NPCType
```

## Functions
```
Domain          Codomain
[               [
    1   ----->      x
    2   ----->      y
    3   ----->      z
                    u
]               ]
        f
```
applies to every single value of the Domain to a value of the CoDomain (the CoDomain may be larger)

```
Domain          Codomain
[               [
    1   ----->      x
    2   ----->      y
    3   ----->      z
    4
]               ]
        f
```
f is not a function : there is no `f(4)`

```
Domain          Codomain
[               [
    1   ----->      x
    2   ----->      y
    3   ------->    z
    4   ----/
]               ]
        f
```
this is ok, f can to the same value of the CoDomain for many values of the Domain


### 3 rules every `function` must have :
1. Totality         (must apply to all values in its Domain, must not return null, must terminate (not loop forever))
2. Determinism      (always make the same association at any point of time, pb ex: Random, Time...)
3. No side effects  (only 1 effect : compute its return value, no other effects (actually except: use CPU, use mem))


### Scalazzi
1. Functions        (only use functions, if it's not a function, don't use it)
2. No poly methods  (don't use methods from AnyRef / Java "Object" etc, don't use methods on polymorphic objects)
3. No null          (never use null)
4. No RTTI          (no Runtime Type information/identification, TypeTags etc...)


## Higher-order functions

An abstraction :
values are cards
functions are rules to play these cards
a program is a game


In Scala, functions are Monomorphic : they cannot take type parameters
`val f: A => B // can't compile, A and B are unknonw, and can't be provided`
Only methods can be Polymorphic :
```
object foo {
  def f[A, B]: A => B
}
```
Methods can take 1 type parameter list, and 0 or more value parameter lists


## Kinds

type of Value is a Type
type of Type is a Kind

List is not a type, it's a type constructor, aka a function from a set of types to a set of List-of-type
```
Domain               Codomain
*                    List[*]
[                    [
    Int      ----->      List[Int]
    String   ----->      List[String]
    Boolean  ----->      List[Boolean]
]                    ]
        List
```
It is applied by calling `[]` instead a `()`

A Kind is the type signature for a Type constructor !

Kind identification exemples :
```
case class Foo[F[_], A](fa: F[A])
// takes 2 type parameters
// Kind: (* => *, *) => *
// F[_]'s kind is * => *
```

Relating value level to type level :
```
val list = List(1, 2, 3)
val plus = (Int, Int) => Int = (a, b) => a + b
val increment = plus(1, _) // partial application
list.map(increment) // using the partial application : map expected a function that takes 1 value param
                    // we can't use plus that takes 2

Map[A, B]
IntMap[Int, B] // partial application of the type constructor
Sized[IntMap] // using it where Sized expected only 1 type parameter
              // same we couldn't use Map that takes 2
```

## Typeclasses

A set of 3 things :
1. Types
2. Operations on values of those types
3. Laws governing the operations

In scala : encoded using `trait`s
1. Types are the Type parameters of the trait
2. Operations are the methods of the trait
3. Laws are... comments in the Scaladoc... (at best some Scalacheck testing...)


Scala look for implicit typeclasses in the Companion object
It's good practice to put it there


## The FP Game :
A card game :
 - the cards are functions, that we can use to make progress, from A to B
 - winning is writing the return value of the function we're trying to write
 - the types are here to constraint us, to limit the cards we can play

 - if we have too many cards we can play, it's bad, our types are not precise enough
 - then it's good to be generic, throw away information that is not meaningful in our context
 - we can also be stuck if we throw away too many information

 - typeclasses are here to help : they precise or add information to our types, hence help us to play the correct cards

