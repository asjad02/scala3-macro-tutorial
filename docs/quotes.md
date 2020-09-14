# Quoted Code

A quoted code block `'{ ... }` is syntactacaly similar to a string quote `" ... "` with the diffecence that the fist contains typed code.
To insert a code into other code we use the `$expr` or `${ expr }` where `expr` is of type `Expr[T]`.
Intuitively, the code directly within the quote is not excecuted now, while the code within the splices is evaluated and their results are then spliced int the sourounding expression.

```scala
val msg = Expr("Hello")
val printHello = '{ print($hello) }
println(printHello.show) // print("Hello")
```

In general, the quote delays the excecution while the splice makes it happen before the sourounding code.
This generalization allows us to also give meeining to a `${ .. }` that is not within a quote, this evaluate the code within the splice at compile-time and place the result in the generated code.
Due to some thechincal considerations we only allow it directly within `inline` definitions that we call a [macro][macros].

It is possible to write a quote within a quote, but usually when we write macros we do not encounter such code.

## Level consitency
One cannot simple write any arbitrary code within quotes and within splices.
A part of the program will live at compile-time and the other will live at runtime.
Consider the folllowing ill-consturucted code.

```scala
def myBadCounter1(using QuoteContext): Expr[Int] = {
  var x = 0
  '{ x += 1; x }
}
```
The problem with this code is that `x` exists durring compilation, but then we try to use it after the compiler has finished (maybe even in another machine).
Clearly it would be impossible to access its value and update it.

Now consider the dual version, where we define the variable at runtime and try to access it at compile-time.
```scala
def myBadCounter2(using QuoteContext): Expr[Int] = '{
  var x = 0
  ${ x += 1; 'x }
}
```
Clearly, this should work as the variable does not exist yet.
To make sure you can only write programs that do not contain these kinds of probems we restrict the set of references to variable and other definitions.

We introduce _levels_ as a count of the number of quotes minus the number of splices surrounding an expression or definition.

```scala
// level 0
'{ // level 1
  var x = 0
  ${ // level 0
    x += 1 
    'x // level 1 
  }
}
```

The system will allow at any level references to global definitions such as `println`, but will restrict refrences to local definitions.
A local definition can only be accessed if it is defined a the same level as its reference.
This will catch the errors in `myBadCounter1` and `myBadCounter2`.

Eventhoug we cannot rever to variable inside of a quote, we can still pass its current value to it by lifting the value to an expression using `Expr.apply`.


## Generics

When using type parameters or other kinds of abstract types with quoted code we will need to keep track of some of these types explicitly.
Scala uses erased-types semantics for its generics.
This implies that types are removed from the program when compiling and the runtime does not have to track all types at runtime.

Consider the following code
```scala
def evalAndUse[T](x: Expr[T]) = '{
  val x2: T = $x // error
  ... // use x2
}
```

Here we will get an error telling us that we are missing a contextual `Type[T]`.
Therefore we can easely fix it by writing
```scala
def evalAndUse[X](x: Expr[X])(using Type[X])(using QuoteContext) = '{
  val x2: X = $x
  ... // use x2
}
```
This code will be equivalent to the more verbose
```scala
def evalAndUse[X](x: Expr[X])(using t: Type[X])(using QuoteContext) = '{
  val x2: t.T = $x
  ... // use x2
}
```
Note that `Type` has a type member called `T` that refers to the type held within the `Type`, in this case `t.T` is `X`.
Note that even if we used it implicitly is better to keep it contextual as some changes inside the quote may require it.
The less verbose version is usually the best way to write the types as it is much simpler to read.
In some cases, we will not know statically the type within the `Type` and will need to use the `.T` to refer to it.

When do we need this extra `Type` parameter?
* When a type is abstract and it is used in a level that is larger than the current level.

When you add a `Type` contextual parameter to a method you will either get it from another context parameter or implicitly with a call to `Type.apply`.
```scala
evalAndUse(Expr(3))
// is equivalent to
evalAndUse[Int](Expr(3))(using Type[Int])
```
As you may have guessed, not every type is can be used in this `Type[..]` out of the box.
We cannot recover abstract types that have already been erased.
```scala
def evalAndUse[T](x: Expr[T])(using QuoteContext) =
  given Type[T] = Type[T] // error
  '{
    val x2: T = $x
    ... // use x2
  }
```

But we can write more complex types that depend on these abstract types.
For example, if we look for or construct explicitly a `Type[List[T]]`, then the system will require a `Type[T]` in the current context to compile.

Good code should only add `Type` to the context parameters and never use them explicitly.
Explicit use is useful while debugging at the cost of conciseness and clarity.


## Liftables
The `Expr.apply` method uses intances of `Liftable` to perform the lifting.
```scala
object Expr:
  def apply[T](x: T)(using QuoteContext, Liftable[T]): Expr[T] =
    summon[Liftable[T]].toExpr(x)
```

`Liftable` is defined as follows:
```scala
trait Liftable[T]:
  def toExpr(x: T): QuoteContext ?=> Expr[T]
```

The `toExpr` method will take a value `T` and generate code that will construct a copy of this value at runtime.

We can define our own `Liftable`s like:
```scala
given Liftable[Boolean] = new Liftable[Boolean] {
  def toExpr(x: Boolean) =
    if x then '{true}
    else '{false}
}

given Liftable[StringContext] = new Liftable[StringContext] {
  def toExpr(x: StringContext) =
    val parts = Varargs(stringContext.parts.map(Expr(_)))
    '{ StringContext($parts: _*) }
}
```
The `Varargs` constructor just creates an `Expr[Seq[T]]` which we can efficiently splice as a varargs.
In general any sequence can be spliced with `$mySeq: _*` to splice it a varargs.

## Quoted patterns
Quotes can also be used to check if an expression is equivalent to another or deconstruct an expression into it parts.


### Matching exact expression

The simples thing we can do is to check if an expression matches another know expression.
Bellow we show how we can match some expressions using `case '{...} =>`

```scala
def valueOfBoolean(x: Expr[Boolean])(using QuoteContext): Option[Boolean] =
  x match
    case '{ true } => Some(true)
    case '{ false } => Some(false)
    case _ => None

def valueOfBooleanOption(x: Expr[Option[Boolean]])(using QuoteContext): Option[Option[Boolean]] =
  x match
    case '{ Some(true) } => Some(Some(true))
    case '{ Some(false) } => Some(Some(false))
    case '{ None } => Some(None)
    case _ => None
```

### Matching partial expression

To make thing more compact, we can also match patially the expression using a `$` to match arbitrarry code and extract it.

```scala
def valueOfBooleanOption(x: Expr[Option[Boolean]])(using QuoteContext): Option[Option[Boolean]] =
  x match
    case '{ Some($boolExpr) } => Some(valueOfBoolean(boolExpr))
    case '{ None } => Some(None)
    case _ => None
```

### Matching types of expression

We can also match agains code of an arbitrarry type `T`.
Bellow we match agains `$x` of type `T` and we get out an `x` of type `Expr[T]`.

```scala
def exprOfOption[T: Type](x: Expr[Option[T]])(using QuoteContext): Option[Expr[T]] =
  x match
    case '{ Some($x) } => Some(x) // x: Expr[T]
    case '{ None } => Some(None)
    case _ => None
```

We can also check for the type of an expression

```scala
def valueOf(x: Expr[Any])(using QuoteContext): Option[Any] =
  x match
    case '{ $x: Boolean } => valueOfBoolean(x) // x: Expr[Boolean]
    case '{ $x: Option[Boolean] }  => valueOfBooleanOption(x) // x: Expr[Option[Boolean]]
    case _ => None
```
Or similarly for an some subexpression

```scala
case '{ Some($x: Boolean) } => // x: Expr[Boolean]
```

### Matching reciver of methods

When we want to match the reciver of a method we need to explicitly state its type

```scala
case '{ ($ls: List[Int]).sum } =>
```

If we would have written `$ls.sum` we would not have been able to know the type of `ls` and which `sum` method we are calling.

Another common case where we need type annotations is for infix operations.
```scala
case '{ ($x: Int) + ($y: Int) } =>
case '{ ($x: Double) + ($y: Double) } =>
case ...
```

### Matching function expressions

*Coming soon*

### Matching types

*Coming soon*

## Unliftables

The `Expr.unlift`, `Expr.unlift.orError` `Unlifted.unapply` method uses intances of `Unliftable` to perform the unlifting.
```scala
extension [T](expr: Expr[T]):
  def unlift(using QuoteContext)(using unlift: Unliftable[T]): Option[T] =
    unlift(expr)

  def unliftOrError(using QuoteContext)(using unlift: Unliftable[T]): T =
    unlift(expr).getOrElse(eport.throwError("...", expr))
end extension

object Unlifted:
  def unapply[T](expr: Expr[T])(using QuoteContext)(using unlift: Unliftable[T]): Option[T] =
    unlift(expr)
```

`Unliftable` is defined as follows:
```scala
trait Unliftable[T]:
  def fromExpr(x: Expr[T])(using QuoteContext): Option[T]
```

The `toExpr` method will take a value `T` and generate code that will construct a copy of this value at runtime.

We can define our own `Uniftable`s like:
```scala
given Unliftable[Boolean] = new Unliftable[Boolean] {
  def fromExpr(x: Expr[Boolean])(using QuoteContext): Option[Boolean] =
    x match
      case '{ true } => Some(true)
      case '{ false } => Some(false)
      case _ => None
}

given Unliftable[StringContext] = new Unliftable[StringContext] {
  def fromExpr(x: Expr[StringContext])(using qctx: QuoteContext): Option[StringContext] = x match {
    case '{ new StringContext(${Varargs(Consts(args))}: _*) } => Some(StringContext(args: _*))
    case '{     StringContext(${Varargs(Consts(args))}: _*) } => Some(StringContext(args: _*))
    case _ => None
  }
}
```
Note that we handled two cases for the `StringContext`.
As it is a `case class` it can be created with the `new StringContext` or with the `StringContext.apply` in the companion object.
We also used the `Varargs` extractor to match the arguments of type `Expr[Seq[String]]` into a `Seq[Expr[String]]`.
Then we used the `Consts` to match known constants in the `Seq[Expr[String]]` to get a `Seq[String]`.


## The QuoteContext
The `QuoteContext` is the main entry point for the creation of all quotes.
This contex is usually just passed around through contextual abstractions (`using` and `?=>`).
Each quote scope will provide have its own `QuoteContext`.
New scopes are intoduced each time a splice is intoduced `${...}`.
Though it looks like a splice takes an expression as argument, it actually takes a `QuoteContext ?=> Expr[T]`.
Therfore we could actually write it explicitly as `${ (using q) => ... }`, this might be useful when debuging to avoid generated names for these scopes.

The method `scala.quoted.qctx` provides a simple way to use the current `QuoteContext` without naming it.
It is usually imported along with the `QuoteContext` using `import scala.quoted._`.

```scala
${ (using q1) => body(using q1) }
// equivalent to
${ body(using qctx) }
```
If you explicitly name a `QuoteContext` `qctx` you will shadow this definition.

When we write a top level splice in a macro we are calling something similar to the following definition.
This splice will provide the initial `QuoteContext` asociated with the macro expansion.
```scala
def $[T](x: QuoteContext ?=> Expr[T]): T = ...
```

When we have a splice within a quote, the inner quote context will depend on the outer one.
This link is represented using the `QuoteContext.Nested` type.
This relation tells us that it is safe to use expressions createed with `q1` within the scope of `q2` but not the other way around (this constraint is statically checked yet).

```scala
def f(using q1: QuoteContext) = '{
  ${ (using q2: q1.Nested) ?=>
      ...
  }
}
```

We can imagine that a nested splice is like the following method, where `ctx` is the context recived by the sourounding quote.
```scala
def $[T](using ctx: QuoteContext)(x: ctx.Nested ?=> Expr[T]): T = ...
```

## β-reduction
When we have a lambda applied to an argument in a quote `'{ ((x: Int) => x + x)(y) }` we do not reduce it within the quote, the code is kept as is.
There is an optimization that β-reduce all lambdas directly applied to parameters to aboid the creation of the closure.
This will not be visible from the quotes perspective.

Sometime it is useful to perform this β-reduction on the quotes directly.
We provide the function `Expr.betaReduce[T]` that recives an `Expr[T]` and β-reduce if it ccontains a directly applied lambda.

```scala
Expr.betaReduce('{ ((x: Int) => x + x)(y) }) // returns '{ val x = y; x + x }
```


## Summon values
There are two ways to summon values in a macro.
The first is to have a `using` parameter in the inline method that is passed explicitly to the macro implementation.

```scala
inline def setFor[T](using ord: Ordering[T]): Set[T] =
  ${ setForCode[T]('ord) }

def setForCode[T: Type](ord: Expr[Ordering[T]])(using QuoteContext): Expr[Set[T]] =
  '{ TreeSet.empty[T](using $ord) }
```

In this scenario, the context parameter is found before the macro is expanded.
If not found, the macro will not expand.

The second way is using `Expr.summon`.
This allows to programatically search for distinct given expressions.
The following example is similar to the previous example.

```scala
inline def setFor[T]: Set[T] =
  ${ setForCode[T] }

def setForCode[T: Type](using QuoteContext): Expr[Set[T]] =
  import scala.collection.immutable._
  Expr.summon[Ordering[T]] match
    case Some(ord) => '{ TreeSet.empty[T](using $ord) }
    case _ => '{ HashSet.empty[T] }
```

The difference is that in this sencario we do start expanding the maro before the implicit search failure and we can write arbirtary code to hande the case where it is not found.
Here we used `HashSet` and another valid implemetation that does not need the `Ordering`.


⮕ [Continue to TASTy Reflection][tasty]

[best-practices]: /docs/best-practices.md
[compiletime]: /docs/compiletime.md
[cross-compilation]: /docs/cross-compilation.md
[faq]: /docs/faq.md
[inline]: /docs/inline.md
[macros]: /docs/macros.md
[migration-status]: /docs/migration-status.md
[quotes]: /docs/quotes.md
[references]: /docs/references.md
[tasty]: /docs/tasty-reflection.md
