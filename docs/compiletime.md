---
id: compile-time-operations
title: Compile-time operations
---

[Scala docs: scala.compiletime][compiletime-api]

## Reporting

It is possible to emmit error messages when inlining code.

```scala
inline def doSomething(inline mode: Boolean): Unit = 
  if mode then ...
  else if !mode then ...
  else error("Mode must be a know value")

doSomething(true)
doSomething(false)
val bool: Boolean = ...
doSomething(bool) // error: Mode must be a know value
```

If `error` is called ouside an inline method the error will be emitted when compiling that call.
If the `error` is written inside an inline method, the error will be emitted only if after inlining the call in not removed as part of a dead branch.
In the previous example we used the value of `mode` is know we would only keep one of the first two branches.

If we want to include part the source code of the arguments in the error message we can use the `code` string interpolator.

```scala
inline def doSomething(inline mode: Boolean): Unit = 
  if mode then ...
  else if !mode then ...
  else error(code"Mode must be a know value but got: $mode")

val bool: Boolean = ...
doSomething(bool) // error: Mode must be a know value but got: bool
```

### Summoning

There are two ways to summon values in inline methods, the first is with a `using` parameter and the second is with one of `summonInline`, `summonAll` or `summonFrom`.
`using` will summon the value at call site before inlining as if the method was not `inline`.
On the other hand, `summonInline` will summon after inlining if the call is not eliminated form a dead branch.
Summon all provides a way to summon multiple values at the same time from a tuple type.
`summonFrom` provides a way to try several implicit searches.

## Values
* `constValue`, `constValueOpt` and `constValueTuple`
* `S`
*Coming soon*

## Testing
* `testing.typeChecks` and `testing.typeCheckErrors`

## Assertions
* `byName`

*Coming soon*

## Inline Matching
* `erasedValue`

*Coming soon*

## Ops

### `scala.compiletime.ops.boolean`
*Coming soon*

### `scala.compiletime.ops.int`
*Coming soon*

### `scala.compiletime.ops.int`
*Coming soon*

### `scala.compiletime.ops.any`
*Coming soon*


⮕ [Continue to Scala 3 macros][macros]


[best-practices]: best-practices.md
[compiletime]: compiletime.md
[cross-compilation]: cross-compilation.md
[faq]: faq.md
[inline]: inline.md
[macros]: macros.md
[migration-status]: https://scalacenter.github.io/scala-3-migration-guide/docs/macros/migration-status.html
[quotes]: quotes.md
[references]: references.md
[tasty]: tasty-reflection.md
[compiletime-api]: https://dotty.epfl.ch/api/scala/compiletime/index.html
