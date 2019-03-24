---
title: "Go: The Diamond Problem"
date: 2017-07-12
keywords:
- go
- golang
- diamond problem
- go diamond problem
---

I've witnessed some of the online debate when it comes to Go and OOP. Some people say the language _is_ object-oriented,
others don't share that view. I think it's safe to say that Go isn't a _traditional_ object-oriented language.

Lately, I've been trying to learn more about Go from an OOP perspective - It's what I have most experience in. In fact,
all of the languages I have worked with on a professional setting follow typical OOP standards (until now!).

After diving into structs and interfaces, as well as concepts like embedding (inheritance?), I was left wondering:
How does Go solve the [diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)?

> The "diamond problem" is an ambiguity that arises when two classes B and C inherit from A, and class D inherits
from both B and C. If there is a method in A that B and C have overridden, and D does not override it, then which
version of the method does D inherit: that of B, or that of C?

Ok, so Wikipedia has a pretty clear explanation on how Go handles it:

> Go prevents the diamond problem at compile time. If a structure D embeds two structures B and C which both have a
method F(), thus satisfying an interface A, the compiler will complain about an "ambiguous selector" if D.F() is
called, or if an instance of D is assigned to a variable of type A. B and C's methods can be called
explicitly with D.B.F() or D.C.F().

Essentially, Go 'solves' the problem at compile time: It just won't allow you to do it.
The method has to be called explicitly or not at all.

Personally, I think this is a good solution. It avoids ambiguity and confusing code - There is already plenty of that
going around!

A practical example (also available at [play.golang.org](https://play.golang.org/p/DmKYcZkeIj)

```go
package main

import "fmt"

type A interface {
    F()
}

type B struct {
    A
}

func (b B) F(){
    fmt.Println("F() of B")
}

type C struct{
    A
}

func (c C) F(){
    fmt.Println("F() of C")
}


type D struct {
    B
    C
}

func main(){
    d := D{}

    // Error! "ambiguous selector d.F".
    // d.F()

    // Allowed!
    d.B.F()
    d.C.F()
}
```