---
layout: sip
title: SIP-NN - for with control flow
vote-status: pending
permalink: /sips/:title.html
redirect_from: /sips/pending/for-with-control-flow.html
---

**By: Yang, Bo**

## History

| Date          | Version       |
|---------------|---------------|
| Jul 20th 2021 | Initial Draft |

## Motivation

Coroutines (including generators and async functions), were considered as a killer feature, but now become a usual practice, after being adopted in ECMAScript and other mainstream languages. Due to lack of runtime support in JVM, all previous implementations of this feature in Scala world, including [Scala Continuations](https://github.com/scala/scala-continuations), [Scala Async](https://docs.scala-lang.org/sips/async.html), [Each](https://github.com/ThoughtWorksInc/each), [Monadless](https://github.com/monadless/monadless), [Dsl.scala](https://github.com/ThoughtWorksInc/Dsl.scala), are all based on compile time CPS translation.

Unfortunately, the exact translation rules have never been standardized. The former [SIP-22 Async](https://docs.scala-lang.org/sips/async.html) contains an Async Transform Specification section, which is too general to tell how should an implementation behave. This proposal is one of a series of proposals, aiming to standardize the exact rules that turn existing `for` comprehension into a full featured continuation, covering not only all the use cases of coroutines in mainstream languages, but also repeatable continuations including data-binding and reactive event observables.

This proposal enables the generator operator `<-` to appear not only as a direct child statement of a `for` loop or a `for` comprehension, but also in any block, as long as the block is a descendant node of a `for`.

## Motivating Examples

Suppose you are creating a function to fetch the website of a Github repository, which is the home page of the repository if available, or the owner's blog site otherwise. With the help of this proposal, you can create something like this:

``` scala
import org.scalajs.dom.ext.Ajax
import scala.scalajs.js.JSON
import scala.concurrent._
def findWebSite(repositorySlug: String) = {
  for {
    repositoryResponse <- Ajax.get(s"https://api.github.com/repos/$repositorySlug")
    repository = JSON.parse(repositoryResponse.responseText)
    homepage = repository.homepage
    webSite = if (homepage != null) {
      homepage.toString()
    } else {
      ownerResponse <- Ajax.get(repository.owner.url.toString())
      owner = JSON.parse(ownerResponse.responseText)
      owner.blog.toString()
    }
  } yield webSite
}
```

According to this proposal, blocks that contain `<-` should be translated to a `for` expression, while control flow keywords that contain `<-` should be converted to function calls. Finally, the above code should be translated as the following code:

``` scala
def findWebSite(repositorySlug: String) = {
  for {
    repositoryResponse <- Ajax.get(s"https://api.github.com/repos/$repositorySlug")
    repository = JSON.parse(repositoryResponse.responseText)
    homepage = (repository: JSON.parse).homepage
    webSite <- keywords.ifThenElse(
      keywords.pure(homepage != null),
      keywords.pure {
        homepage.toString()
      },
      for {
        ownerResponse <- Ajax.get(repository.owner.url.toString())
        owner = JSON.parse(ownerResponse.responseText).owner
      } yield owner.blog.toString()
    )
  } yield webSite
}
```

This proposal also includes some runtime library requirements:

``` scala
package scala.concurrent

object keywords {

  def pure[A](a: => A)(using ExecutionContext) = Future(a)

  def ifThenElse[A](
    ifFuture: Future[Boolean],
    thenFuture: Future[A],
    elseFuture: Future[A],
  )(using ExecutionContext) = {
    ifFuture.flatMap { b => if (b) thenFuture else elseFuture }
  }

  // Other runtime libraries for each Scala control flow
  // ...
}
```

With the help of the runtime libraries, the return type of `findWebSite` will eventually be inferred as a `Future[String]`.

## Design

### Virtualization

[Scala Virtualization](https://github.com/TiarkRompf/scala-virtualized/) was a previous attempt to translate Scala control flow to function calls solved by name instead of by symbol. Unfortunately, Scala Virtualization does not take CPS translation into account, as a result, it conflicts with Scala Continuation and Scala Async.

This proposal extends the idea of virtualization to CPS translation. Therefore, by import different implementations of `keywords` object, `for` with control flow should work for various of types. Also it's trivial to create a `keywords` object that delegates calls to a generic type class, like `MonadError`.

### Reifiable control flow

The implementation of `keywords` can be `case class`es that preserve the structure of control flow, both at type level and runtime. For example:

``` scala
trait Mappable[Keyword, +Value]

def[Upstream, Value, Mapped, MappedValue](
  upstream: Upstream
)flatMap(
  using Mappable[Upstream, Value]
)(
  flatMapper: Value => Mapped
)(
  using Mappable[Mapped, MappedValue]
) : keywords.FlatMap[Upstream, Value, Mapped] =
  keywords.FlatMap(upstream, flatMapper)

def[Upstream, Value, Mapped, MappedValue](
  upstream: Upstream
)map(
  using Mappable[Upstream, Value]
)(
  mapper: Value => Mapped
)(
  using Mappable[Upstream, Value]
) : keywords.Map[Upstream, Value, Mapped] =
  keywords.Map(upstream, mapper)

object keywords {
  case class IfThenElse[If, Then, Else](ifTree: If, thenTree: Then, elseTree: Else)
  export IfThenElse.{apply => ifThenElse}
  given[If, Then, Else, Value](using thenMappable: Mappable[Then, Value], elseMappable: Mappable[Else, Value]) as Mappable[IfThenElse[If, Then, Else], Value]

  case class Pure[A](a: () => A)
  def pure[A](a: A) = Pure(() => a)
  given[A] as Mappable[Pure[A], A]

  final case class FlatMap[Upstream, UpstreamValue, Mapped](
    upstream: Upstream,
    flatMapper: UpstreamValue => Mapped
  )
  given[Upstream, UpstreamValue, Mapped, Value](using Mappable[Mapped, Value]) as Mappable[FlatMap[Upstream, UpstreamValue, Mapped], Value]
  final case class Map[Upstream, UpstreamValue, Mapped](
    upstream: Upstream,
    mapper: UpstreamValue => Mapped
  )
  given[Upstream, UpstreamValue, Mapped] as Mappable[Map[Upstream, UpstreamValue, Mapped], Mapped]

  // Other control flow ASTs
  // ...
}

object Ajax {
  trait Response {
    def responseText: String
  }
  case class get[Url](url: Url)
  given[Url] as Mappable[Ajax.get[Url], Response]
}


import scala.language.dynamics
case class SelectDynamic[Parent, Field <: String & Singleton](parent: Parent, field: Field) extends Dynamic
def [Parent <: Dynamic](parent: Parent)selectDynamic(field: String) = SelectDynamic[Parent, field.type](parent, field)
object JSON {
  case class parse(json: String) extends Dynamic
}
```

If we import the above definitions of `keywords`, `JSON` and `Ajax` instead of `scala.concurrent.keywords`, `scala.scalajs.js.JSON`, and `org.scalajs.dom.ext.Ajax`, then the `findWebSite` function should return a reified tree of the following type:

``` scala
keywords.FlatMap[
  keywords.Map[Ajax.get[String], Ajax.Response, (Ajax.Response, JSON.parse, 
    SelectDynamic[JSON.parse, "homepage"]
  )],
  (Ajax.Response, JSON.parse, SelectDynamic[JSON.parse, "homepage"]), 
  keywords.Map[
    keywords.IfThenElse[keywords.Pure[Boolean], keywords.Pure[String], 
      keywords.Map[
        keywords.Map[Ajax.get[String], Ajax.Response, (Ajax.Response, 
          SelectDynamic[JSON.parse, "owner"]
        )],
        (Ajax.Response, SelectDynamic[JSON.parse, "owner"]),
        String
      ]
    ],
    String,
    String
  ]
]
```

It's a tree of control flow, which can then interpreted by type-level interpreters.

### Tagged partial functions

When control flow is reified, each `case` clause in pattern matching or exception handling might produce different reified types. In case of inferring them to `Any` type, the list of `case` clauses will be translated to a partial function whose return value is wrapped in a couple of `keywords.right` and `keywords.left` calls. The number of `keywords.right` calls indicates the index of the case clause. For example:

```
def matchWebSite(repositorySlug: String) = {
  for {
    repositoryResponse <- Ajax.get(s"https://api.github.com/repos/$repositorySlug")
    repository = JSON.parse(repositoryResponse.responseText)
    homepage = repository.homepage
    webSite = homepage match {
      case null =>
        ownerResponse <- Ajax.get(repository.owner.url.toString())
        owner = JSON.parse(ownerResponse.responseText)
        owner.blog.toString()
      case nonNullHomepage =>
        nonNullHomepage.toString()
    }
  } yield webSite
}
```

will be translate to

```
def matchWebSite(repositorySlug: String) = {
  for {
    repositoryResponse <- Ajax.get(s"https://api.github.com/repos/$repositorySlug")
    repository = JSON.parse(repositoryResponse.responseText)
    homepage = (repository: JSON.parse).homepage
    webSite <- keywords.matchCase(
      keywords.pure(homepage),
      {
        case null =>
          keywords.left(
            for {
              ownerResponse <- Ajax.get(repository.owner.url.toString())
              owner = JSON.parse(ownerResponse.responseText).owner
            } yield owner.blog.toString()
          )
        case nonNullHomepage =>
          keywords.right(keywords.left(keywords.pure(nonNullHomepage.toString())))
      },
    )
  } yield webSite
}
```

For naive `keywords` implementation `keywords.right` and `keywords.left` can be simply `identity` function. However, for `keywords` implementation that reifies control flow to data structures that preserves everything at type-level, `keywords.right` and `keywords.left` can be implemented as a coproduct or `scala.Either`.

## Specification

### DSL expressions

A DSL expression is a DSL control flow expression or a DSL block.

A DSL control flow expression is a control flow expression, which could be a `try`, `catch`, `finally`, `if`, `else`, `match`, `case`, `do` or `while` but neither a `for` loop nor a `for` comprehension, and one or more of its direct child expressions are DSL expressions.

A DSL block is block expression, and one or more of its direct child expressions are generator operators `<-`, DSL blocks or DSL control flow expressions.

To create DSL expressions, the grammar of blocks should be changed to:

``` ebnf
Block        ::=  BlockStat {semi BlockStat} [ResultExpr]
               |  Enumerators

Enumerators  ::=  Enumerator {semi Enumerator}

Enumerator   ::=  Generator
               |  Expr1
```
For example:

``` scala
// This is a DSL block:
{
  b <- a
  c <- b
  f(c)
}
```

``` scala
// This is a DSL control flow expression:
while (x) {
  b <- a
  c <- b
  f(c)
}
```

A DSL expression must eventually belongs to a `for` loop or a `for` comprehension, or it will not compile.

``` scala
// Should not compile
def f(s: Seq[Int]) = {
  while (scala.util.Random.nextBoolean()) {
    a <- s
    println(a)
  }
}
```

### Translation rules

The following translation rules should be added to existing `for` expression translation procedure:

* A value definition `p = e` inside a `for` loop or `for` comprehension expression, where `e` is a DSL expression, is translated to `p <- wrap(e)`, where *wrap* is an internal AST translation process  (see below).
* A generator `p <- e` inside a `for` loop or `for` comprehension expression, where `e` is a DSL expression, is translated to `p' <- wrap(e); p <- p'`.
* A `for` comprehension `for (p <- e) yield e'`, where `e'` is a DSL expression, should raise a compile error, unless the translation rule is specified in another proposal, which could be *Rebindable `for`*.
* A `for` loop `for (p <- e) do e'`, where `e'` is a DSL expression, should raise a compile error, unless the translation rule is specified in another proposal, which could be the *Rebindable `for`*.
* `wrap(if (e1) e2 else e3)` is expanded to `keywords.ifThenElse(wrap(e1), wrap(e2), wrap(e3))`
* `wrap(do e1 while e2)` is expanded to `keywords.doWhile(wrap(e1), wrap(e2))`
* `wrap(while e1 do e2)` is expanded to `keywords.whileDo(wrap(e1), wrap(e2))`
* `wrap(e1 match e2)` is expanded to `keywords.matchCase(wrap(e1), wrapCaseList(e2))`, where *wrapCaseList* is an internal AST translation process (see below).
* `wrap(try e1 finally e2)` is expanded to `keywords.tryFinally(wrap(e1), wrap(e2))`
* `wrap(try e1 catch e2 finally e3)` is expanded to `keywords.tryCatchFinally(wrap(e1), wrapCaseList(e2), wrap(e3))`.
* `wrap(try e1 catch e2)` is expanded to `keywords.tryCatch(wrap(e1), wrapCaseList(e2))`.
* `wrap { ...stats; r }` is expanded to `for { ...stats } yield r`.
* `wrap(e)`, where `e` is not a DSL expression, is expanded to `keywords.pure(e)`.
* `wrapCaseList { case p0 => b0 ... case pn => bn }` is expanded to `{ case p0 => keywords.left(wrap(b0)) ... case pn => keywords.right( ... keywords.right(keywords.left(wrap(bn))) ... )) }`, where `n` is the index of each `case` clause, and `keywords.right` repeats `n` times.

## Next steps

### Single block `for`

Both `for` comprehension and `for` loop need two blocks of clause. It would be good if `for` without `yield` is allowed, like this:

``` scala
for {
  b <- a
  c <- b
  c
}
```

However, the above syntax is not included in this proposal. Instead, it should be addressed in another proposal.

### `for` with definitions

Definitions other than generator operator `<-` and value definition `=` in a DSL block are not covered in this proposal. There should be another proposal for `lazy val`, `var`, `def`, `class`, `trait`, `object` definitions. For example:

``` scala
for {
  b <- a
  lazy val d = {
    c <- b
    c
  }
  e <- d
} yield e
```

### !-notation

The generator operator `<-` is verbose. It would be good if we have a conciser syntax:

``` scala
for {
  c = !f(!f(a))
} yield c
```

Which should equal to the following code.

``` scala
for {
  b <- f(a)
  c <- f(b)
} yield c
```

The detail of the !-notation will be discussed in another proposal.

### Rebindable `for`

It's common to use loops or comprehension in an asynchronous context. In ECMAScript, we can create a `for` loop with `await` in an `async` function:

``` javascript
for (const element of collection) {
  await f(element);
}
```

However, Scala `for` comprehension can't. The reason is this functionality requires a different type signature other than `map`/`foreach`, which should be instead the signature similar to `traverse`/`traverseU` in Scalaz or Cats. Unfortunately a Scala `for` comprehension is always translated to `map` on the collection. To allows `for` comprehension in an asynchronous context, we need a mechanism to overload the `map` method defined on the collection. This need will be addressed in another proposal.