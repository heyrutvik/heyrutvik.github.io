---
layout: post
title:  "Properties of Expression"
date:   2018-02-21 18:00:00 +0530
categories: functional
---

Programming languages must provide some means of writing an expression.
An Expression is something that is always evaluates to value.

```
For example, 
1 + 2 
a mathematical expression always evaluates to 3.
```

### properties

_These properties has simple premise with far-reaching implications._

+ Referential Transparency (RT)
	- it means that you can replace any sub-expression with its value in an expression without affecting any behaviour of expression.
	
+ Composition
	- you can compose any number of expressions to create new expression.

RT has very important hidden implication which is removing side-effect. A side-effect means doing anything besides computing and returning value from given expressions. Such as, writing to files or changing data structures in place.

RT helps to reason about code using simple substitution model and Composition helps to solve complex problem by dividing it in sub problems and combining their solution.