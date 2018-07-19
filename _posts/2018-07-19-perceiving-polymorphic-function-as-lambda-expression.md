---
layout: post
title:  "Perceiving Polymorphic Function As Lambda Expression"
date:  2018-07-19 20:00:00 +0530
---

[Note: This is informal post in which I use lambda expression to understand polymorphic function.]

### Lambda expression
From the perspective of a programmers, more pervasive term for [lambda expression](https://en.wikipedia.org/wiki/Lambda_calculus#Definition){:target="_blank"} is an **anonymous function**. So, the lambda expression `(λ(x,y).(x + y))` can be represented in javascript as `(x, y) => { return x + y; }`. Of course, the operation `+` is not available in lambda calculus but [here](https://github.com/heyrutvik/playground/blob/master/js/fizz-buzz.js#L209){:target="_blank"}'s one way to encode it.

### Currying
It is a technique to transform a function that takes multiple arguments into a function of one argument that returns another function as its result.

```javascript
// a function which takes multiple arguments 
let sum = function(x, y) {
  return x + y;
}

// a function which takes any function `f` that takes two parameters 
// and gives us curried form of that `f` 
let curry = function(f) {
  return (x) => {
    return (y) => {
      return f(x, y)
    }
  }
}

// curried form of the function `sum`
let csum = curry(sum)

sum(1, 2) == (csum(1))(2)
```

[Note: In lambda calculus, lambda expressions are always curried.]

### Polymorphic function
A function with more than one **form** can be considered as a polymorphic function. But how one can define the term *form*? If we consider the only languages with static type system then we can define *more than one form* as follows,

> If the same function can compute the result regardless of the **type** of its arguments, then it said to be polymorphic. 

[Note: We will focus here on *parametric polymorphism*, not an ad-hoc polymorphism.] 

For example, let's define a function that gives us the count of elements of given list of integer.

```scala
scala> :pa
// Entering paste mode (ctrl-D to finish)

// count the elements of list of integer
def count(ls: List[Int]): Int = ls.length

val l1 = List(1,2,3) // list of integer

val l2 = List("1", "2", "3") // list of string

// Exiting paste mode, now interpreting.

count: (ls: List[Int])Int
l1: List[Int] = List(1, 2, 3)
l2: List[String] = List(1, 2, 3)

scala> count(l1) // OK
res0: Int = 3

scala> count(l2) // ERROR
<console>:14: error: type mismatch;
 found   : List[String]
 required: List[Int]
       count(l2)
```

Here, the problem with this implementation of `count` is that it will only work with the list of integer. If we try to pass any other *type* of list then compiler will complain about type mismatch. It means, we have to define `count` for each type of list, but the counting is such an operation which has nothing to do with the *type* of the list.

We need some mechanism to **abstract over the type** of list. It's just another way of saying we need a function with many *forms*.

```scala
scala> :pa
// Entering paste mode (ctrl-D to finish)

def gcount[T](ls: List[T]): Int = ls.length

val l1 = List(1,2,3) // list of integer

val l2 = List("1", "2", "3") // list of string

// Exiting paste mode, now interpreting.

gcount: [T](ls: List[T])Int
l1: List[Int] = List(1, 2, 3)
l2: List[String] = List(1, 2, 3)

scala> gcount[Int](l1) // OK
res0: Int = 3

scala> gcount[String](l2) // OK
res1: Int = 3
```

We can think of it as that we are defining the collection of functions - each function for each type - but their implementation is *single* or *same*.

### Understand polymorphic function using curried lambda expression
If we ask any programmer that how many argument does `gcount` function take, then - most probably - answer would be one. Well, that's a correct answer. But for the sake of understanding, let's assume that `gcount` takes two parameter - one the type of the list and another list itself.

We have such mechanism to transform a function with two argument into a function of one argument. We can apply `curry` to `gcount` and generate a new function which is the curried form of `gcount`.

```javascript
// ASSUME that `count` will work only with list of integer :grimacing:
let count = function(ls) {
  return ls.length
}

// define type using javascript Symbol
let Int = Symbol.for('int')
let String = Symbol.for('string')
let Double = Symbol.for('double')

// our type universe only contains Int and String. Double is excluded!!
let types = [Int, String]

// make new polymorphic function `g` from given function `f`
// `g` takes type in first argument and list in second argument.
// `f` should be one argument function
let makePolymorphic = function(f) {
  // collection of functions
  // notice: same function `f` is mapped to each type `t`.
  let collection = types.map((t) => {return {[t]: f}})
  // return a function which takes type
  return (type) => {
    // find mapping from type to function using type
    let mtf = collection.find(s => s[type])
    if (mtf === undefined) { // if given `type` is out of universe
      return "error: function not defined for type;"
    } else {
      // return a function which takes list
      return (list) => {
        // collect `f` by applying `type` to `mtf`
        // seems almost like parametric polymorphism :laughing:
        return mtf[type](list)
      }
    }
  }
}

// `gcount` is a polymorphic version of `count`
let gcount = makePolymorphic(count)

// application
gcount(Int)([1,2,3]) // OK -> 3
gcount(String)(["1","2","3"]) // OK -> 3

// ERROR -> "error: function not defined for given type;"
gcount(Double)

// ERROR ->
//VM77:1 Uncaught TypeError: gcount(...) is not a function
//  at <anonymous>:1:15
gcount(Double)([1.2,2.3,3.4])

```

[Note: Please read code comments, before reading further]

The gist of above code snippet is that it is mimicking the *Scala* implementation of `gcount` in *Javascript*. Javascript is dynamic typed language so we do not have *explicit* type annotation on source code level. So, we are using javascript `Symbol` to represent types in our example. We are also *assuming* that types are [first-class citizen](https://en.wikipedia.org/wiki/First-class_citizen){:target="_blank"} so we can [pass them around as value](http://docs.idris-lang.org/en/latest/tutorial/typesfuns.html#dependent-types){:target="_blank"}.

The function `makePolymorphic` takes any function `f` and returns a lambda expression which takes *type*. If the given type is exist in our assumed universe then it will return another lambda expression which takes list and computes the answer. If the given type is out of our universe then we are simply return error string. That's why when we try to apply list to `gcount(Double)`, it's throwing error that  `gcount(...) is not a function`.

By comparing following two lines, one may have a clear idea of what's going on here.

in Scala,
```scala
gcount[Int](List(1,2,3))
```

in Javascript,
```javascript
gcount(Int)([1,2,3])
```

Final remark, polymorphic function can be seen as a lambda expression which takes *type* as first argument and return a function which takes a list of previously given *type*.

---

### Afterthought

In the same spirit, we can also understand **Type constructor** - meaning is self explanatory. It's *something* which construct a new type. We can say that a function is **Value constructor** which construct a new value. The [`List`](https://www.scala-lang.org/api/current/scala/collection/immutable/List.html){:target="_blank"} in above example is type constructor. Which we can define roughly as follows,

```scala
sealed trait List[T]
final case class Nil[T]() extends List[T]
final case class Cons[T](h: T, t: List[T]) extends List[T]
```

[Note: `Nil` and `Cons` are data constructor.]

 We can imagine the `List` as a function from `type` to `type`.  We feed any concrete type to `List` and construct a new type. So in short, `List` in itself is not a type, when you write `List[Int]` or `List[String]` then it is a type. In same manner, [`Either`](https://www.scala-lang.org/api/current/scala/util/Either.html){:target="_blank"} is also a type constructor. It takes two concrete types and construct a new type - kinda function with two arguments to generate a new type. `Either[Int, String]` or `Either[String, Int]` are types, not the `Either` itself.