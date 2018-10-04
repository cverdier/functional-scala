# Day 2

## Semigroup

Elements can be appended
```
trait Semigroup[A] {
  def append(l: => A, r: => A): A
}
```
Law of the Semigroup :
 - Associativity :
   ```
   append(x, append(y, z)) === append(append(x, y), z)
   x |+| (y |+| z) === (x |+| y) |+| z
   ```

For Int for example, you can have a Addition Semigroup or a Multiplication Semigroup.
What do you use as implicit ?
To solve this, we can use `NewTypes`

```
case class Sum(value: Int)

```

There are many more for Int (or others):
 - FirstSemigroup (always take the first of 2)
 - LastSemigroup (always take the last of 2)
 - MaximumSemigroup
 - MinimumSemigroup
 ...


## Monoid

Has a Zero value
```
trait Monoid[A] {
  def mzero: A
}
```
Laws of the Monoid
 - law of the Semigroup
 - the zero value has no effect on the append operation
 ```
 append(mzero, a) === a
 append(a, mzero) === a
 ```

## Functor

```
trait Functor[F[_]] {

  def map[A, B](fa: F[A])(f: A => B): F[B]

  def fmap[A, B](f: A => B): F[A] => F[B]
}
```
It takes a type parameter of Kind '* => *'. That already tells us what may be a Functor and what cannot
(ex String is *, cannot; Option is * => *, may; Set is * => *, may; Map is (*, *) => * cannot but, may if partially applied)

Laws (better understood with `fmap` syntax) :
 - Identity:    `fmap(identity) === identity`
 - Composition: `fmap(f.compose(g))) == fmap(f).compose(fmap(g))`

Set is not a Functor : it breaks the composition law because it distincts elements :
 A => C (distinct) may not be equivalent to A => B (distinct) => C (distinct)

/!\ Functors are not data structures
(though most Data structures are functors)
ex: Future
ex: a function :
 - I have a function A => ? (partially applied: I know the type A)
 - I can have a functor A => B
 - It will allow me to go from A => ? to B => ? (via function compisition)

Functors can be seen as :
 - Programs with F-shaped instructions sets
 - Recipes (cooking recipes) and how you mix them is determined by F
 (cf picture)


Madness : Nothing can have any Kind !!
hence we can have a functor of Nothing (assuming Kind * here)
This is useful to allow define other typeclasses. Ex: Functor or Eq for Either[Nothing, A] ...


### Zip

zip takes a Product of F[_]s, and turns them into a F[Product]

```
def zipOption[A, B](l: Option[A], r: Option[B]): Option[(A, B)] =
(l, r) match {
  case (Some(a), Some(b)) => Some((a, b))
  case _ => None
}

def zipWith[A, B, C](l: Option[A], r: Option[B])(f: ((A, B)) => C): Option[C] =
zipOption(l, r).map(f)

def zipList1[A, B](l: List[A], r: List[B]): List[(A, B)] =
(l, r) match {
  case (a :: as, bs) => zipList1(as, bs) ++ bs.map(b => (a, b))
  case (Nil, _) => Nil
}
def zipList2[A, B](l: List[A], r: List[B]): List[(A, B)] =
(l, r) match {
  case (a :: as, b :: bs) => (a, b) :: zipList2(as, bs)
  case (Nil, _) => Nil
}
```

### Apply / Zip

Apply is applying a function inside an F to a value inside an F to obtain a new value inside an F
```
trait Apply[F[_]] extends Functor[F] {
  def ap[A, B](fa: F[A])(fab: F[A => B]): F[B]
}
```
(ap is useful on curried functions, not easy to use in scala, way better in haskell were all functions are curried)

Zip is combining as a tuple, inside an F, two values each inside an F
Apply can be defined using Zip
```
trait Apply[F[_]] extends Functor[F] {
  def ap[A, B](fa: F[A])(fab: F[A => B]): F[B] =
    map(zip(fa, fab)){ case (a, ab) => ab(a) }

  def zip[A, B](l: F[A], r: F[B]): F[(A, B)]
}
```

keep left / keep right
```
implicit class ApplySyntax[F[_], A](l: F[A]) {
  def *> [B](r: F[B])(implicit F: Apply[F]): F[B] =
    F.zip(l, r).map(_._2)

  def <* [B](r: F[B])(implicit F: Apply[F]): F[A] =
    F.zip(l, r).map(_._1)
}
```
can be useful with Future for ex: both must succeed, but we'll keep only one of the result
warning :
```
fa <* fb !=== fa
fa *> fb !=== fb
```

ex:
```
val l = List(1, 2, 3)
val r = List(9, 2)

// with the cartesian cross-product zip
val lr1 = List((1, 9), (1, 2), (2, 9), (2, 2), (3, 9), (3, 2))

// with the index-align zip
val lr2 = List((1, 9), (2, 2))

val lr1_mapped1 = lr1.map(_._1) // List(1, 1, 2, 2, 3, 3)   (l <* r) !== l
val lr1_mapped2 = lr1.map(_._2) // List(9, 2, 9, 2, 9, 2)   (l *> r) !== r
val lr2_mapped1 = lr2.map(_._1) // List(1, 2)
val lr2_mapped2 = lr2.map(_._2) // List(9, 2)
```

### Applicative

```
trait Applicative[F[_]] extends Apply[F] {
  def point[A](a: => A): F[A]
}
```
point aka pure : pure lifting a a value to an F
not doing anything to the structure of F
point will always succeed, never 'halt', always produce F having the a (Some for Option, Success for Try...)

properties:
```
fa <* point(b) === fa
point(b) *> fa === fa
```
remember:
```
fa <* fb !=== fa
fa *> fb !=== fb
```

exemples :
 - Future.successful(_)
 - Success(_)
 - Some(_)
 _ List(_) // 1 element
 ...


Example of something that is Apply but not Applicative :
`Map[K, ?]` (partially applied)
 - you can write 'ap'
 - you can't write point, because you can't create the key K


## Monad

```
trait Monad[F[_]] extends Applicative[F] {
  def bind[A, B](fa: F[A])(f: A => F[B]): F[B]
}
```

Operations:
```
  option.flatMap((a: A) => ???: Option[B]): Option[B]

```

Remember :
 - fa must run before f
 - f may depend the value from fa, and produce more 'F's, and can do depending on the context

```
  implicit val OptionMonad: Monad[Option] =
    new Monad[Option] {
      override def point[A](a: => A): Option[A] = OptionApplicative.point(a)
      override def ap[A, B](fa: => Option[A])(f: => Option[A => B]): Option[B] = OptionApplicative.ap(fa)(f)

      override def bind[A, B](fa: Option[A])(f: A => Option[B]): Option[B] =
        fa match {
          case None => None
          case Some(a) => f(a)
        }
    }

  implicit val ListMonad: Monad[List] =
    new Monad[List] {
      override def bind[A, B](fa: List[A])(f: A => List[B]): List[B] =
        fa match {
          case Nil => Nil
          case a :: as => f(a) ++ bind(as)(f)
        }

      override def point[A](a: => A): List[A] = ???
      override def ap[A, B](fa: => List[A])(f: => List[A => B]): List[B] = ???
    }
```

Comparision to imperative programming :
```
a = doX()
b = doY(a)
c = doZ(a, b)
if (b > c) doW(b, c) else doU(a)
```

Monads = sequential computation, purely functionnal :)


Difference between Zip and Monad :
 - with Zip you may have sequenciallity (exemple: the Parses we used)
 - but with Zip the 'r' cannot depend on the 'a' produced by 'l', whereas with Monad it can

Monad: context-sensitive grammar


## Summary : the Functor hierarchy
Functor - Gives us the power to map values produced by programs without changing their structure
Apply - Adds the power to combine two programs into one by combining their values
Applicative - Adds the power to produce a 'pure' program that produces the given result
(Bind - // adds the 'bind' operation, but not all the laws of Monad)
Monad - Adds the power to feed the result of one program into a function, which can look at the runtime value and return a new program, which is used to produce the result of the bind

(there are a few more that add more structure to monads)
monad plus, alternative ...

// Semigroupal == Apply (it's a different definition)



## Foldable

(cf scalaz)
```
trait Foldable[F[_]] {
  def foldRight
  def foldMap
}
```
Functionnal way to iterate over something (collections... any * => *)


## Traverse

(cf scalaz)

useful operations (I mean, superpower operations !)
 - sequence // like Future.sequence -- it's like swap the type parameters !
 - traverse : map then sequence // effectful loop


## Optics

```
type Optic[S, A]
```
S - super-structure
A - sub-structure
an Optic S A allows you to focus in on a sub-structure A inside of a super-structure S, for purpose of accessing or modifying A

type of optics:
Lenses : focus in a term in a Product type
Prism : focus in a term in a Sum type

```
case class Lens[S, A](
  get: S => A,
  set: A => (S, S)
) {

}
```

Interesting : the traversal optic, to update values over a collection for ex

