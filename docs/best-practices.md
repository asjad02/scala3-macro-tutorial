# Best practices

## Inline
**Coming soon**

## Macros
**Coming soon**

## Quoted code

### Keep quotes readable
* Try to avoid `${..}` with arbitrary expressions inside
  * Use `$someExpr`
  * Use `${ someExprFrom('localExpr) }`

To ilustrate, consider the following example:
```scala
val x: StringContext = ...
'{ StringContext(${Varargs(stringContext.parts.map(Expr(_)))}: _*) }
```
Insead we can write the following:

```scala
val x: StringContext = ...
val partExprs = stringContext.parts.map(Expr(_))
val partsExpr = Varargs(partExprs)
'{ StringContext($partsExpr: _*) }
```
The contents of the quote are cleared this way.

### Avoid nested contexts

Consider the following code:

```scala
val y: Expr[Int] = ...
def body(x: Expr[Int])(using qctx.Nested) =  '{ $x + $y }
'{ (x: Int) => ${ body('x) } }
```

Instead, use a normal context and pass all needed expressions.
This has also the advantage of allowing the funtion to not be defined locally.
```scala
def body(x: Expr[Int], y: Expr[Int])(using QuoteContext) =
  '{ $x + $y }

val y: Expr[Int] = ...
'{ (x: Int) => ${ body('x, y) } }
```



## TASTy reflection
**Coming soon**
