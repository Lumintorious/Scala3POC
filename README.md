Hello, I wanted to prove that a typesafe effect system could be built without over-reliance on flatmaps and for comprehensions, but in natural ways using methods/functions, this way, the paradigm isn't forced on outside libraries (Java, Kotlin don't need to be turned into IOs in your project)

Using Context functions, Poly functions and eventually erased values + safer exceptions

```Scala
@main def testOwnEffect =
 val effect = UEffect[IOException]: // Unsafe Effect that might fail with IOException
   ~Terminal.putLine(s"What is your name?")
   val input = ~Terminal.readLine() // You can run effects within effects with `~`
   println(s"Hello, ${input}") // You can mix and match normal unit effects
   input

  // You can run effects outside effects only with a Runner
  // Effects provide their Runner to children inside using Context parameters
  // Nesting is possible, but stack safety isn't, although there are ways around that
  Runner().run(effect) match 
    case Right(str) => ...
    case Left(exception) => ...
    
@main def testZIOEffect = // There is a 1-to-1 mapping as you can see
  val effect =
    for
      _ <- console.putStrLn("What is your name?")
      input <- console.getStrLn()
      _ <- console.PutStrLn(s"Hello, ${input}")
    yield input

  Runtime.default.unsafeRunSync(effect) match
    case Success(_) => ...
    case Failure(_) => ...
```

Implementation (Excuse the mess, it's just a poc):

```Scala
final class CanThrow[-T <: Throwable] private()
object CanThrow:
  def make[E <: Throwable] = new CanThrow[E]
  def apply[E <: Throwable] = [X] => (block: CanThrow[E] ?=> X) => (c: CanThrow[E]) ?=> block(using c)

infix type ?![X, E <: Throwable] = CanThrow[E] ?=> X

class Runner:
  inline def run[E <: Throwable, T](eff: Effect[T ?! E]): Either[E, T] =
    try
      given CanThrow[E] = CanThrow.make[E]
      Right(eff.unsafeRun(using this))
    catch
      case ex: E => Left(ex)

class Effect[+T](either: => Runner ?=>  T):
  def unsafeRun(using Runner): T = either
  def unary_~(using Runner): T  = either

object Effect:
  def apply[E <: Throwable, T](op: => Runner ?=> (T ?! E)): Effect[T ?! E] =
    new Effect[T ?! E]( (_: Runner) ?=> (_: CanThrow[E]) ?=> op )

class UEffect[E <: Throwable]:
  def apply[T](op: => Runner ?=> (T ?! E)): Effect[T ?! E] =
    new Effect[T ?! E]( (_: Runner) ?=> (_: CanThrow[E]) ?=> op )

object UEffect:
  def apply[E <: Throwable] = new UEffect[E]
	
object Terminal:
  def readLine(): Effect[String ?! IOException] = UEffect[IOException]:
    throw IOException("Console corrupted.")

  def putLine(str: String): Effect[Unit ?! IOException] = UEffect[IOException]:
    println(str)
```

I'm glad to answer any questions and take any constructive feedback.
