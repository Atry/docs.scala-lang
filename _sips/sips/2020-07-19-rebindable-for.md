---
layout: sip
title: SIP-NN - Overloadable for
vote-status: pending
permalink: /sips/:title.html
redirect_from: /sips/pending/overloadable-for.html
---

**By: Yang, Bo**

## History

| Date          | Version       |
|---------------|---------------|
| Jul 19th 2020 | Initial Draft |

## Motivation


## Concepts
A DSL expression is a DSL `for` expression, a DSL control flow expression or a DSL block.

A DSL `for` expression is a `for`/`yield` comprehension whose `yield` clause is a DSL expression, or a `for`/`do` loop, whose `do` clause is a DSL expression.

``` scala
// This is a DSL for expression 
for (b <- a) yield {
  c <- b
  c
}
```

``` scala
// This is a normal for comprehension, not a DSL for expression 
for (b <- a; c <- b) yield {
  c
}