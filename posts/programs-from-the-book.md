---
title: Programs from THE BOOK
date: '2013-09-17'
description:
categories:

type: draft
---

This is 
I wanted to post a link to the presentation below on Google+ and write a short
comment, but

[http://www.infoq.com/presentations/polymorphism-functional-oop](http://www.infoq.com/presentations/polymorphism-functional-oop)

A short glimpse into the polymorphism facilities of Clojure, OCaml, Haskell
and Scala, and how they solve the expression problem. Personal comments below.


Clojure:

* ++ Nice, short and properly namespaced, but...
* - ...could probably use some more documentation which is normally implied by
  the type annotations of typed languages.
* - With no explicit module passing or type inference, constants like
  'identity' are expressed as functions from a representative value of the
  type, which is slightly hackish and leads to some spam (the 'head' method in
  the example code).
* ? The example implementation is quite wonky. A reasonable way of
  implementing it would be using nil instead of :tip, extending Foldable to it
  and removing the bizarre extensions of Addable and Foldable to nil and
  Object, respectively.


OCaml:

* -- Structural (duck) typing of module interfaces. Implies a global shared
  namespace for all interface members (e.g. ADDABLE is TYPE by virtue of using
  the type named t).
* ++ Explicit passing of modules makes eval independent of type inference and
  therefore programs read more clearly.


Haskell:

* - A lot of the type parameters are often implicit. The evaluator therefore
  needs to subsume the type inferencer, which makes programs harder to read.
* ? Not much syntactic distinction between instantiating a type class for a
  (polymorphic) type vs for a type constructor (i.e. functor).
* ++ Sexy and succinct notation.


Scala:

* ++ Instance declarations reified as objects and given names.
* + Implicit objects allow for polymorphism inference as in Haskell (though it
  seems weaker from the example).
* ? I don't know Scala enough to be able to spot more weirdness/greatness.


This presentation reminded me of a question that has been nagging for me the
last few days, so I'm asking all the typed people here. :-)

Is there some theorem, observation, evidence or other reason to belief that in
a good type system, there should be only one reasonable implementation of a
function given its type?

E.g., should all monads that wrap their results in lists behave as the List
monad? Same for product and union and the Writer and Error monads,
respectively. Is the standard List monad implementation the program from The
Book that proves the type-as-formula?

I have seen at least one example of people using type systems
