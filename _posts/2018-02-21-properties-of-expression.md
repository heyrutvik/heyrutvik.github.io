---
layout: post
title:  "Properties of Expression"
date:  2018-02-21 22:00:00 +0530
---

The notion of an **expression** is  central in functional programming (FP) and all the other features of FP can be boils down to it. An Expression is something that is always evaluates to value.

```
For example, 
1 + 2 
a mathematical expression always evaluates to 3.
```

### properties

+ Referential Transparency (RT)
	- It means that the same expression, given the same values as input, will always return the same output in any context, as in math.
	- RT has very important hidden implication which is removing side-effect. A side-effect means doing anything besides providing value using given inputs. Such as, writing to files or changing data structures in place. It helps you to reason about code using simple substitution model.

+ Composition
	- You can compose any number of expressions to create new expression.
	- It helps to solve complex problem by dividing it in sub problems and combining their solutions.
	
These properties has simple premise with far-reaching implications.