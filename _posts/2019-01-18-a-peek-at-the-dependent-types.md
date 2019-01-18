---
layout: post
title:  "A Peek At The Dependent Types"
date:  2019-01-19 00:00:00 +0530
---

## Preface

I have been reading about dependent types on and off for almost six months. I have been longing to write about it for someone who knows nothing about it[^1], and for another reason[^2].

## Familiar language constructs

To me, a language is a functional if function can be treated as a value. A function is a value which means that I can store it in a variable, pass it to another function as argument or return it from another function. So, function is not a special syntax of language but belongs to a set of expressions.

If one thinks about it, it means that a function is an expression which depends on _another expression_. The another expression is called parameter.


Ex. in Scala
```scala
val inc: Int => Int = _ + 1
```
Ex. in Idris
```idris
inc: Int -> Int
inc x = x + 1
```

Here, `inc` expecting _another expression_ of type `Int` and gives an expression of type `Int`. So, it's an example of "**expression depending on expression**".

If one adds parametric polymorphism into above capability, then one can express an expression which returns an expression of different type, according to provided type.

Ex. in Scala[^3]
```scala
def id[A](x: A): A = x
```
Ex. in Idris
```idris
id : a -> a
id x = x
```

Here, `id` is expecting an expression of _any_ type and gives an expression of the (given) same type. So, it's an example of "**expression depending on type**".

If one adds a capability to define the generic type, then we can have a language in which a construct can take _another type_ to form a new type.


Ex. in Scala
```scala
trait List[A]
case class Cons[A](h: A, t: List[A]) extends List[A]
case object Nil extends List[Nothing]
```
Ex. in Idris
```idris
data List : (elem : Type) -> Type where
  Nil : List elem
  (::) : (h : elem) -> (t : List elem) -> List elem
```

Here, `List` is a type constructor which takes a type and build a new type. It's an example of "**type depending on type**". So, `List[Int]`is type which depends on `Int`.

These are familiar concepts which we encounter in most mainstream languages. The only combination of "expression" and "type" we haven't discussed yet is "**type depending on expression**". If one thinks about it for a moment, will realize that it's a bit strange idea. We used to think about the **type** as a special construct which language provides to avoid some kind of mistakes or to help compiler writers to optimize our code according to types. It is okay to think that an expression depends on the type but other way around? Hmm.. To make this possible, we have to bring down types to the level of expressions so that we can manipulate a type according to some expression.

## The world where types are as common as expressions

The phrase "types are as common as expressions" means that I can manipulate type in the same way as I manipulate expressions. So, I can store it in the variable, pass or return them from functions as well as _I can form a type which may contain an expression_.

Let's look at example so those words can make more sense. We already have defined a `List` structure which can hold any number of element of the same type. `List[Int]` only describes that it has elements of type `Int`, but it tells us nothing about the length of it. If I want to describe a length in type, I need an ability to define type which depends on expression. In our case, that expression would be of type `Nat` (for natural number, we don't want a length in negative) which describes a length of data direct in its type. We already used the name `List`, so let's call it `Vect`. `Vect` is same as `List` but `Vect` has an ability to describe the length of the data in its type.

 Ex. in Idris
```idris
data Nat = Z | S Nat

data Vect : (n : Nat) -> (elem : Type) -> Type where
  Nil : Vect Z elem
  (::) : (h : elem) -> (t : Vect n elem) -> Vect (S n) elem
```

Observe the different between the type of `List` and `Vect`. `List` is `Type -> Type` where as `Vect` is `Nat -> Type -> Type`. So, `Vect` needs an expression as its first argument and type as its second argument and builds a type.

### A Type depends on an expression is called a dependent type.[^4]

Even if our language provides such ability, what is it good for us? First thing come to my mind, is that we can describe our intention of the program in the type (directly) as much as possible. A plus point of that would be that compiler will _help_ us to check whether our definition matches our intention. An example of that would make more sense.

If one is familiar with functional language, then I can bet he will also be familiar with `map` function. Let's define `map` on `List`.

```idris
map : (a -> b) -> List a -> List b
map f [] = []
map f (x :: xs) = f x :: map f xs

Idris> map (+ 1) [1,2,3]
[2, 3, 4] : List Integer
```

Above code snippet will work as expected as expected, but if someone defines the `map` as shown in following code snippet, compiler wouldn't be able to help us at compile time.

```idris
map : (a -> b) -> List a -> List b
map f xs = []

Idris> map (+ 1) [1,2,3]
[] : List Integer
```

Above code snippet is type correct, but it doesn't match our intention and compiler can't help us because we haven't described our intention in the type of the `map`. Let's try to define `map` on `Vect` type.

```idris
map : (a -> b) -> Vect n a -> Vect n b
map f [] = []
map f (x :: xs) = f x :: map f xs

Idris> map (+ 1) [1,2,3]
[2, 3, 4] : Vect 3 Integer
```

Notice, the type of `[2, 3, 4]` is `Vect 3 Integer`, where `3` is total elements and `Integer` is type of elements. In above snippet, type of `map` is `(a -> b) -> Vect n a -> Vect n b` which means compiler knows that the length of returned vector should be same `n` as input vector's. So, it won't allow us to make similar mistake we did in `List`'s `map` above.

```idris
map : (a -> b) -> Vect n a -> Vect n b
map f xs = []

  |
6 | map f xs = []
  |            ~~
When checking right hand side of map with expected type
        Vect n b

Type mismatch between
        Vect 0 b (Type of [])
and
        Vect n b (Expected type)

Specifically:
        Type mismatch between
                0
        and
                n
```

Idris will let us to know that our intention is to return a vector with `n` elements but we are returning a vector with `0` elements. Expected type is `Vect n b` but type of `[]` is `Vect 0 b` and it won't compile. In short, compiler won't even let us define a _foolish_ definition of a program if it knows more about types (or in another words, about our intentions).

## Postface

This was only the tip of the iceberg. There is much more interesting and mind expanding things about the dependent types but the title of this post says "**a peek at**" for a reason. Those things will may be in another post.

<br />
<hr />
[^1]: I assume, one is familiar to typed functional language.
[^2]: I want to know how much I understood after all the attempts.
[^3]: `def` is special construct to define method. It isn't a value but bear with me for the sake of demonstration.
[^4]: Well, that was as easy as eating a Π... isn't?
