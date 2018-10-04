# Day 5

## Local Effects

// See MonadTransformers in exercices

// Another approach
```
  type MyStateType
  type Task[A] = IO[Throwable, A]

  class MyIO[A](val run: Task[A]) extends AnyVal

  def createMonadState[S]: Task[MonadState[Task, S]] = ???

  def myLocalState[F[_]: MonadState[?, MyStateType]]: F[Boolean] = ???

  for {
    monadState <- createMonadState[MyStateType]
    result     <- myLocalState[MyIO](monadState) // pass the implicit typeclass explicitlty here
  } yield result

```

In Scala, FreeMonads/FreeStructures are way better than MonadTransformers (more perf !)
MonadTransformers are a Product of Effects, while FreeMonads allow to do Sum of Effects

-> see ScalaZ 8 for a FreeMonad to replace stacking of MonadTransformers


## Typeclasses organization

1. Each typeclass represents some business subdomain, e.g account credits/debits
2. All methods on the typeclass relate to each other through algebraic or operational laws
   `transact(debit(account, amount) *> credit(account, amount)) === balance(account)`
3. Delete all methods that don't relate to each others through algebraic or operational laws
4. All operations should be able to be expressed in terms of a small set of orthogonal operations
5. It's a good sign if you can express one type class in terms of another, lower-level one


`override` is a sign of trouble, OOP machinery
`extends` too, except to model typeclasses


## Avoiding inhenritance for code organization / code architecture

```
def myGetName[F[_]: Console: Logging: Monad]: F[String] =
  for {
    _ <- Console.putStrLn("Hello, what is your name?")
    _ <- Logging[F].log("My log line")
    n <- Console.getStrLn
    _ <- Console.putStrLn("Good to meet you, " + n + "!")
  } yield n

// this implem here doesn't have to know that F requires Console &  Logging
def myThirdPartyCode[F[_]: Monad](getName: F[String]): F[Unit] = ???

// at this higher level, when call myThirdPartyCode here, we'll have to provide Console, Logging & Monad
myThirdPartyCode(myGetName)

// but we kept myThirdPartyCode's implem ignorant of that
```


## Free (Monad)
```
// you get a Monad for "free" from a Functor, you don't have to implement it as a monad
sealed trait Free[F[_], A]
case class Return[F[_], A](a: A) extends Free[F, A]
case class Join[F[_], A](fa: F[Free[F, A]]) extends Free[F, A]

// ex:
sealed trait ConsoleF[A]
case class ReadLineOp[A](next: String => A) extends ConsoleF[A]
case class WriteLineOp[A](line: String, next: A) extends ConsoleF[A]

// you build a big structure of operations as a Sum type extenfing Free
// then it can be interpreted with 'fold'
sealed trait Free[F[_], A] {
  def interpret[G[_](f: F ~> G): G[A]  // aka fold
}
val myProgramIO: IO[Throwable, String] = myProgram.interpret()
// passing here a function that converts every operation to IO : the interpreter
// then running those IO
// so more than 2x performance overhead !
```

Final Tagless is basically better (no perf impact)
But they are combinable for some cases

Use cases for Free
 - interesting if you add more things to Free (than just beeing a Monad)
 - escape some Effects, more efficiently than stacking MonadTransformers



## ??

Start thinking Product ans Sum types for APIs ;)

- Other types of Functor (selectable Functor)
- Other types of Monad (co-monad) // less usefull
- High-order Abstract Syntax (HOAS) : encode a type-safe DSL in scala // a way to do better Spark without Serialization
- Recursion schemes : eliminate some duplication when doing recursion on some types of data structures, and have composable data


## Selectable Functor

```
case class Parser[+E, +A](
  run: String => Either[E, (String, A)]
)
```
this kind of Parser is slow :
 - we always split gigantic strings
 - we need to recur / trampoline

this is very powerful, because it's a Monad
but actually for Parsing we don't need that power

with an Applicative :
```
sealed trait Parser[+E, +A]
case class Fail[E](error: E) extends Parser[E, Nothing]
case class Success[A](value: A)
```


## Variance tricks

if we have Type[+A] we cannot define a function which has A as input type without adding in the parameter of that function [A1 >: A]
if we have Type[-A] we cannot define a function which returns A without adding in the parameter of that function [A1 <: A]


##  Kleisli

Kleisli F : A => F[B]
Kleisli IO : A => F[IO]
normal function : A => B

Kleisli is basically an effectful function

```
val printLine: KleisliIO[IOException, String, Unit] = KleisliIO.impure(???)
val readLine: KleisliIO[IOException, Unit, String] = KleisliIO.impure(???)
readLine >>> printLine // composition
```
KleisliIO.impure(???) does not wrap the code, so when composing both instructions are kept together => this is faster !


