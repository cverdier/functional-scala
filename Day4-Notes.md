# Day 4

How to use ZIO
 - scalaz App
 - unsafeRun were you can
 - where using CatsEffect, mapping via the typeclasses
   -> with http4s, doobie...

Application ideas :

1. A simple functionnal game : console input/ouput and randoms. Ex: The Hanged Man
2. A simulation : simulate an Elevator system in a hotel - they have to be efficient...
3. A web crawler : collect links, follow them, with real http pages (more realistic : more potential for failures)


## The web crawler

- A set of seed Urls
- for each url
   - retrieve the HTML content of the URL
   - handle failures gracefully
   - parse the HTML content // cheating with regex here
   - feed the content and url to a processor : ex index or count words in the page
   - identify all the href links
   - filter the href links according to some criteria
   - update the working set of urls to include the href links
   - include throttling to avoid rate-limiting


## Under-implem notes

```
type Processor[A] = (URL, String) => IO[Nothing, A]
def crawl[A](seeds: Set[URL], processor: Processor[A]): IO[Exception, List[A]
```
We're tempted to return a List[A] here, but we're assuming too much at the wrong place:
it's the Processor who should decide, we just need here to know that there is some structure on A
As should need to be combinable and have a zero value (empty) : hence Monoid
```
type Processor[A] = (URL, String) => IO[Nothing, A]
def crawl[A: Monoid](seeds: Set[URL], processor: Processor[A]): IO[Exception, A]
```

```
final case class Crawl[E, A](errors: List[ProcessorError[E]], value: A)
// vs
final case class Crawl[E, A: Monoid](errors: List[ProcessorError[E]], value: A)
```
We should not constraint data structures, but functions instead :
Crawl can be constructed without knowing A is a Monoid, why prevent that ?
It's when it's used in a function that this restriction appears
Further:
```
final case class Crawl[E, A](error: E, value: A)
object Crawl {
  implicit def CrawlMonoid[E: Monoid, A: Monoid]: Monoid[Crawl[E, A]] =
    new Monoid[Crawl[E, A]] {
      override def zero: Crawl[E, A] = Crawl(mzero[E], mzero[A])
      override def append(l: Crawl[E, A], r: => Crawl[E, A]): Crawl[E, A] = Crawl(l.error |+| r.error, l.value |+| r.value)
    }
}
```
Here we make Crawl a Monoid when E and A are Monoids
This doesn't constraint to build Crawl (it's just a Tuple)
But we can take advantage of this when using it in a function that has Monoid constraints


We didn't loose any capability by going from List[ProcessorError[E]] to E: Monoid in crawl :
We can regain them :
```
 def crawl[E: Monoid, A: Monoid](
    seeds: Set[URL],
    processor: Processor[E, A],
  ): IO[Exception, Crawl[E, A]] = {

    def process1(url: URL, html: String): IO[Nothing, Crawl[E, A]] =
      processor(url, html).redeemPure(
        e => Crawl(e, mzero[A]),
        a => Crawl(mzero[E], a)
      )


    ???
  }

  def crawlE[E, A: Monoid](
    seeds: Set[URL],
    processor: (URL, String) => IO[E, A]
  ): IO[Exception, Crawl[List[ProcessorError[E]], A]] = {

    crawl(
      seeds,
      (url: URL, html: String) => processor(url, html).redeem(
        e => IO.fail(List(ProcessorError(e, url, html))),
        a => IO.now(a)
      )
    )
  }
```

Reduce a List of Monoids :
```
def reduceAll[A: Monoid](list: List[A]): A =
  list match {
    case Nil => mzero[A]
    case a :: as => a |±| reduceAll(as)
  }
```
This is already provided by `fold` or `foldMap` ;)


Stack-safe recursion :
use an accumulator parameter on the "loop" method, to be able to put it in tail position


```
def sum(list: List[Int]): Int = list match {
  case Nil => 0
  case i :: is => i + sum(is) // (i + ???) operation is on the stack while ??? is computed, each step adds
}

def sum(list: List[Int]): IO[Nothing, Int] = list match {
  case Nil => IO.now(0)
  case i :: is => sum(is).map(_ + i) // .map(_ + i) is trampolined by IO, but will consumer Heap
}

@tailrec // will convert to a while loop
def sum(list: List[Int], acc: Int): Int = list match {
  case Nil => acc
  case i :: is => sum(is, acc + i) // now in tail position
}

// no tailrec annotation in this case ;)
def sum(list: List[Int], acc: Int): IO[Nothing, Int] = list match {
  case Nil => IO.now(acc)
  case i :: is => sum(is, acc + i) // tail position : nothings needs to be saved on the Heap to be computed after
}
```


A technique for Retry (and other things) :
 - create a New Type for IO (ex MyIO)
 - HttpClient's typeclass implem for MyIO uses the retry strategy for MyIO
 -> so you "select" to have a Retry strategy (and which to use) by using MyIO as the F :)



### Monad Transformer

Data types that transform Monads, that add capabilities to Monads

```
case class ErrorT[F[_], +E, +A](run: F[Either[E, A]] {
  def map...
  def flatMap
}
object ErrorT {
  def point...
}

type ErrorfulList[A] = ErrorT[List, Exception, A]
```
ErrorfulList adds the error-handling effect to List
(inside we'll have a List[Either[E, A]]

Usefull for certain effects :
 - Errors
 - State (state monad)
 - config reading (reader monad)
 - Option, Either...

Better used with Final Tagless, so you wrap them in typeclasses
ex :
```
trait MonadError[F[_], E] {
   def fail[A](e: E): F[A]
   def attempt[A](fa: F[A]): F[Either[E, A]]
}

def myCode1: ErrorT[Future, Error, Unit] = ???
def myCode2[F[_]: MonadError[Error, ?]]: F[Unit] = ??? // use this : added capability > wrapping type !!
```

This the MTL style (Monad Transformer Library)


```
// from
def myCode1: Future[Try[Boolean]]

// to
trait FromFuture[F[_]] {
  def fromFuture[A](fa: => Future[A]): F[A]
}
def myCode2[F[_]: MonadError[?, Throwable]: FromFuture]: F[Boolean]

implicit val MyInstance: FromFuture[Future[Try[?]]] with MonadError[Future[Try[?]], Throwable]

implicit val MyInstance2: FromFuture[IO[Throwable, ?] with MonadError[IO[Throwable, ?], Throwable]

```