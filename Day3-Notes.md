# Day 3

Moving from :
 - println(line: String): Unit
 - readLine: String
(those are not functions, they have side-effects)
To versions that return monadic values, so we can build on


In FP, we write Programs as data-structures, that can combine etc
They are only executed at the edge of the application were the effects will actually happen


## ZIO

```
type IO[E, A]
```
IO will produce either en error `E` or a single value `A`
-> error at the end of the world


ZIO has no dependencies
-> there are other packages for interpreters ?

ZIO.seq is "sequential zip"
ZIO.seqWith is seq followed by map
There is a Parallel version too, that's why it's not called zip ;)


attempt method :
```
IO[E, A] => IO[Nothing, Either[E, A]]
```


### From real to FP world

Capturing effects :
```
IO.sync
```
(with laziness)

Capturing effects with exceptions :
```
IO.syncException
IO.syncThrowable
IO.syncCatch // PartialFunction to choose/map what you catch
```
seems not to be using NonFatal :(

IO.absolve:
```
IO[E, Either[E, A]] => IO[E, A]
```

### Fibers

ZIO's implementation of green threads

```
  for {
    queue <- Queue.bounded[String](100)
    fiber <- queue.take.flatMap(putStrLn).forever // this return IO[Nothing, Nothing], and will suspend here forever
    _     <- queue.offer("Hello").forever         // so this line can never be executed
  } yield ???
```

```
  for {
    queue    <- Queue.bounded[String](100)
    consumer <- queue.take.flatMap(putStrLn).forever.fork // now forked on a fiber
    producer <- queue.offer("Hello").forever.fork         // and this one in another
    _        <- consumer.interrupt // fiber can be interrupted
    _        <- producer.join      // fiber can be joined, to await for its value (non-blocking of course)
  } yield ???
```

### Supervision : prevent Thread leaks
```
IO.supervise {
  ... // any Fiber forked here will be terminated when the enclosing IO is completed
}
```
other syntax :
```
myIO.supervised
```

Another approach : fork0
```
fibonacci(100).fork0(errors => if (???) fibonacci(100).map(_ => ()) else IO.unit)
```
allows to specify a handler in case of interruption, and choose what to do, producing a new IO
allows to retry/restart for ex


Fibers can be combined with zip
 - join waits for both results
 - interrupt interrupts both


### Timeout

races the IO with a timeout one and you "fold"-like to produce what you want
```
interrupted1.timeout(None: Option[Unit])(Some(_))(60.seconds)
```

### Resources

ensuring
```
putStrLn("Hello").ensuring(putStrLn("Goodbye"))...
```
this ensures that if the "Hello" IO


Brackets:
IO.bracket
 - acquire, which produces an A   (may fail,     uninterruptable)
 - release, which releases the A  (may not fail, uninterruptable)
 - use, which uses the A          (may fail)

Usage:
```
val acquire: IO[_, A]
acquire.bracket(release(_)) { resource =>
  for {
    bytes <- resource.bytes
  }
}
```


### Run

### Schedule

Every `Schedule[A, B]` can:
 - start with some initial state
 - for every step on the schedule :
   - update its state
   - consume input values of type A
   - produce output values of type B
   - decide to continue or complete the schedule
   - decide to delay the next step by some duration

Possibilities:
 - flexible retry (you can define your own Schedules)

```
(io: IO[E, A]).retry(s: Schedule[A, B]): IO[E, A]
(io: IO[E, A]).retryOrElse(...): IO[E, A] // see doc
(io: IO[E, A]).repeat(s: Schedule[A, B]): IO[E, B]
(io: IO[E, A]).repeatOrElse(...): IO[E, B] // see doc
```

Schedule are Applicative functors