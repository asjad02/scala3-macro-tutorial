---
id: tasty-reflection
title: TASTy Reflection
---

The reflection API provides a more complex and comprehensive view on the structure of the code.
It provides a view on the *Typed Abstract Syntax Trees* **TASTy** and their properties such as types, symbols, positions and comments.

## How to use the API

Accessing this API need and import that depends the current `QuoteContext`.
We can use `scala.quoted.qctx` to import it.

```scala
def pow(x: Expr[Int])(using QuoteContext): Expr[Int] = {
  import qctx.tasty._ // Import Tree, Type, Symbol, Position, .....
  ...
}
```

This will import all the types and modules (with extension methods) of the API.

The full imported API can be found here: [Reflection](http://dotty.epfl.ch/api/scala/tasty/Reflection.html)

For example to find what is a `Term`, we can see in the hireachy that it is a subtype of `Statement` which is a subtype of `Tree`.
If we look into the [`TermMethods`](http://dotty.epfl.ch/api/scala/tasty/Reflection/TermMethods.html) we will find all the extension methods that are defined for `Term` such as `Term.tpe` which returns a `Type`.
As it is a subtype of `Tree` we can also look into the [`TreeMethods`](http://dotty.epfl.ch/api/scala/tasty/Reflection/TreeMethods.html) to find more methods such as `Tree.pos`.
Each type also a module with some _static-ish_ methods, for example in the [TypeModule](http://dotty.epfl.ch/api/scala/tasty/Reflection/TypeModule.html) we can find the method `Type.of[T]` with will create an instance of `Type` containing `T`.


## Relation with expressions
<!-- Term vs Expr -->
<!-- Safty -->
*Coming soon*


## Examples
*Coming soon*
