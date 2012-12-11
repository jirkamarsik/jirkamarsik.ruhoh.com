---
title: Type Raising and the Cooperative Construction of Meaning
date: '2012-12-11'
description: 
categories: computational-linguistics
tags: [semantics, lambda]
---

One of the goals of computational semantics is to find an algorithmic
way of building representations of natural language utterances that
would enable us to perform sound inference on them. Since first-order
predicate logic is a well-studied and expressive formal language, we
will consider the mapping of natural language utterances onto
first-order predicate logic formulas.

In order to build these formulas, we will rely on Frege's
[principle of compositionality] [PoC]. We will assume that our input,
the syntactic representation of the utterance, will be in the form of
a binary constituency tree. We associate a lambda term to every word
of the sentence (a leaf of the constituency tree) using a lexicon. The
semantics of an inner node are then obtained by function application,
with one of the children acting as functor and the other as argument.
We will remark which subconstituent is to be the functor and which the
argument, even though this can be easily inferred from the types of
the lambda terms. The semantic representation of the sentence is
defined as the lambda term associated with the root node.

  [PoC]: http://en.wikipedia.org/wiki/Principle_of_compositionality

Simple Things First
-------------------

Let us consider the simplest possible cases first. We will look at
proper nouns and intransitive verbs.

  * "Vincent" ↦ <span style="color:blue">**vincent**</span> : t
  * "growls" ↦ <span style="color:red">λx. **growl(**x**)**</span> : t → f

In the above lexicon, I use the following typographic conventions:
"double quotes" for natural language expressions, **bold** for the
first-order predicate logic formulas (or their parts) and the
unadorned typeface for the lambda calculus specific matter. Later on,
I will use expressions in double quotes to refer not only to the
expressions themselves, but also to their semantic representations,
either computed through composition or obtained from the lexicon. Note
that the bold parentheses do not correspond to lambda calculus
function applications, they are part of the produced FOPL formulas. I
will always mark lambda applications explicitly with the @ symbol. I
have also taken the liberty to assign types to the lambda terms. The
two atomic types that come naturally in the domain of FOPL formulas
are terms (t) and formulas (f).

The lexicon I show here is very straightforward. We map the proper
noun "Vincent" onto a constant and the intransitive verb "growls" onto
a lambda abstraction which builds an atomic formula by filling the gap
in the unary relation with the supplied argument (we will call
functions that take terms and produce formulas predicates).

Let us take the simple sentence "Vincent growls" and its syntactic
structure.

    (S (NP "Vincent")
       (VP "growls"))

We obtain the semantic representation of the sentence by applying the
"growls" lambda term to the "Vincent" one and β-reducing, <span
style="color:red">**growl(<span
style="color:blue">vincent</span>)**</span> : f.

Let us now consider transitive verbs, as in the sentence "Vincent
likes Mia", with the following lexicon additions

  * "Mia" ↦ <span style="color:green">**mia**</span> : t
  * "likes"
    ↦ <span style="color:red">λx. λy. **like(**y**,** x**)**</span>
    : t → t → f

and syntactic structure.

    (S (NP "Vincent")
       (VP (VT "likes")
           (NP "Mia")))

Unsurprisingly, "Mia" maps to a constant and "likes" maps to a binary
predicate (a function that builds a formula using two terms as
arguments). In the sentence above, "likes" applies to "Mia" first,
yielding <span style="color:red">λy. **like(**y**, <span
style="color:green">mia</span>)**</span> : t → f after β-reduction.
Applying this lambda term to that of "Vincent" yields the final
representation, <span style="color:red">**like(<span
style="color:blue">vincent</span>, <span
style="color:green">mia</span>)**</span> : f.

So far so good. Some things to note about our system so far. First
off, syntactic categories always correspond to a single semantic type
(see table below). Second, the [head constituent] [head] ends up
always being the functor during composition.

  [head]: http://en.wikipedia.org/wiki/Head_(linguistics)

  * S = f
  * NP = t
  * VP = NP → S
  * VT = NP → VP


Quantifiable Complications
--------------------------

We will now generalize our notion of a noun phrase and admit phrases
with determiners such as "a" and "every". The sentence we would like
to be able to analyze is "Every boxer growls". The desired semantic
representation would look something like this **<span
style="color:green">∀x. (<span style="color:blue">boxer(<span
style="color:green">x</span>)</span> → <span
style="color:red">growl(<span
style="color:green">x</span>)</span>)</span>** : f. It is obvious that
the relations **<span style="color:blue">boxer</span>** and **<span
style="color:red">growl</span>** correspond to the words "boxer" and
"growls". What is the semantics of "every" then? Well, as the coloring
hints, it is the quantifier, the implication and the variables.
However, our old way of composing VPs with NPs will not work here, as
one does not simply put a quantifier in the argument position of a
relation.

Here are the new elements of our lexicon.

  * "boxer" ↦ <span style="color:blue">λx. **boxer(**x**)**</span>
    : t → f
  * "every" ↦ <span style="color:green">λP. λQ. **∀x. (**(P@**x**)
    **→** (Q@**x**)**)**</span>
    : (t → f) → (t → f) → f

We represent common nouns as unary predicates (sets of entities are
after all a natural denotation for common nouns). As for the
quantifier, we map it to a function that builds a universal
quantification (see [generalized quantifiers] [GQ]). The first
argument is the restricting predicate which the defines the range of
the variable, the second argument is the predicate which is to be true
in the domain of quantification.

  [GQ]: http://en.wikipedia.org/wiki/Generalized_quantifier

Calculemus! [*](http://en.wikipedia.org/wiki/Gottfried_Wilhelm_Leibniz#Symbolic_thought)

    (S (NP (Det "Every")
           (N "boxer"))
       (VP "growls"))

"Every" applies to "boxer" yielding <span style="color:green">λQ.
**∀x. (**<span style="color:blue">**boxer(<span
style="color:green">x</span>)**</span> **→** (Q@**x**)**)**</span> :
(t → f) → f. The resulting NP then applies to "growls" to produce
**<span style="color:green">∀x. (<span style="color:blue">boxer(<span
style="color:green">x</span>)</span> → <span
style="color:red">growl(<span
style="color:green">x</span>)</span>)</span>** : f. Hurray!

OK, what we have here is not as great as it looks. Contrary to the
previous situations, it is now the subject NP which acts as the
functor, not the VP. However, in our new theory, this is true only for
the case when the NP turns out to be a quantified NP. In the case of a
proper noun, we still make the VP the functor. This slightly annoying
discrepancy has more disturbing implications. The semantic type of one
NP ("every boxer" : (t → f) → f)) can be different from that of another
NP ("Vincent" : t). This loss of consistency between syntactic and
semantic types would make writing the rest of our lexicon difficult as
we would always have to consider both cases whenever dealing with NPs.

Luckily, we can solve this problem by raising the types of the proper
nouns from t to (t → f) → f. Whenever we have lambda terms F : α → β
and G : α, we can construct G' = λP. P@G : (α → β) → β, which
satisfies that F@G = G'@F (proof via simple β-reduction). What it
means is that we can take a functor F and an argument G, do some
trivial change to G, and then use the new G as a functor with F as the
argument while still getting the same result as before. If we apply
this technique to our proper nouns, we get the following updated
lexicon entries.

  * "Vincent" ↦ <span style="color:blue">λP. P@**vincent**</span>
    : (t → f) → f
  * "Mia" ↦ <span style="color:green">λP. P@**mia**</span>
    : (t → f) → f

An intuitive way to look at the current NP might be as a thing that
ranges over some entities and when given a predicate, produces a
formula that is true when the entities being ranged over satisfy the
predicate. Therefore, in the above implementations of "Vincent", the
way to produce a formula which is true whenever <span
style="color:blue">**vincent**</span> satisfies a predicate is to
simply use the constant <span style="color:blue">**vincent**</span> as
the term for the predicate being built.

Stuff Gets Difficult
--------------------

Let's try and do a recap of the types we have so far.

  * S = f
  * N = t → f
  * VP = t → f
  * NP = VP → f
  * VT = ???

Oops, we seem to have a problem. By changing what it means to be an NP
we have broken our transitive verbs which relied on NPs being terms.

If we were to consider a sentence where the transitive verb has a
proper noun as the object (e.g. "Every boxer likes Mia"), things would
still work out (we can type our proper nouns as (t → (t → f)) → (t →
f)). However, when we try to work with a quantified object, things
break.

Let's use the sentence "Every boxer likes a woman" as an example.

    (S (NP (Det "Every")
           (N "boxer"))
       (VP (TV "likes")
           (NP (Det "a")
               (N "woman"))))
        
  * "woman" ↦ <span style="color:teal">λx. **woman(**x**)**</span>
    : t → f
  * "a" ↦ <span style="color:orange">λP. λQ. **∃y. (**(P@**y**) **∧**
    (Q@**y**)**)**</span>
    : (t → f) → (t → f) → f

The semantics of "woman" are straightforward. As for "a", we construct
another quantifier, this time using an existential quantification. We
can construct the meaning of "a woman" as <span
style="color:orange">λQ. **∃y. (**<span
style="color:teal">**woman(<span
style="color:orange">y</span>)**</span> **∧** (Q@**y**)**)**</span> :
(t → f) → f. Now, if we try to apply this lambda term to the one of
"likes" or vice versa, we get nonsense. In the first case, we end up
with a lambda abstraction inside of our FOPL formula, and in the
second case, we place (an abstraction of) a quantification as an
argument of a relation. Neither of these can ever resolve to a proper
formula.

How do we fix this? Well, let us imagine how we would like the meaning
of "likes a woman" to look like.

  * <span style="color:red">λy. <span style="color:orange">**∃y.
  (**<span style="color:teal">**woman(<span
  style="color:orange">y</span>)**</span> **∧** <span
  style="color:red">**like(**y**, <span
  style="color:orange">y</span>)**</span>**)**</span></span> : t → f

The above is one reasonable answer (Beware of the two ys! The orange,
bold one belongs to the constructed FOPL formula and was contributed
by "a". The red, plain one is part of the lambda calculus and should
belong to "like".). These semantics for "likes a woman" look good. If
we apply them to a term, they yield a formula stating that the entity
named by the term does indeed "like a woman", i.e. what we have is a
working predicate.

Now that we know what we want, it is not so difficult to work out the
correct version of the "likes" semantics on paper.

  * "likes"
    ↦ <span style="color:red">λO. λy. O@(λx. **like(**y**,** x**)**)</span>
    : ((t → f) → f) → t → f

OK, that looks good. Let's check if it works.

  * "Every boxer" ↦ <span style="color:green">λQ. **∀x. (**<span
  style="color:blue">**boxer(<span
  style="color:green">x</span>)**</span> **→** (Q@**x**)**)**</span> :
  (t → f) → f
  * "likes a woman" ↦ <span style="color:red">λy. <span
  style="color:orange">**∃y. (**<span style="color:teal">**woman(<span
  style="color:orange">y</span>)**</span> **∧** <span
  style="color:red">**like(**y**, <span
  style="color:orange">y</span>)**</span>**)**</span></span> : t → f
  * "Every boxer"@"likes a woman" ↦ <span style="color:green">**∀x.
  (<span style="color:blue">boxer(<span
  style="color:green">x</span>)</span> → <span style="color:red"><span
  style="color:orange">∃y. (<span style="color:teal">woman(<span
  style="color:orange">y</span>)</span> ∧ <span
  style="color:red">like(<span style="color:green">x</span>, <span
  style="color:orange">y</span>)</span>)</span></span>)**</span> : f

Yay!

We have a consistent theory of the syntax-semantics interface now,
here is our syntax-to-semantics type mapping.

  * S = f
  * N = t → f
  * VP = t → f
  * NP = VP → f
  * VT = NP → VP

The transitive verb is still expressed as a function taking an NP and
producing a VP, which seems somewhat linguistically plausible.


The Part Where I Get Confused Too
---------------------------------

OK, so what did I do to my lexicon entry for "likes" that it magically
started doing exactly what I needed instead of breaking horribly?

If we look at the type signature of our new "likes" representation and
compare it with the old one, we can see that we have swapped out the
old type of NP (t) with its new raised type ((t → f) → f). In the body
of the abstraction, we are calling our argument O and passing it
something which looks like our original value. Isn't this just like
type raising? Why did the meaning of our sentence change then?

Well, it's not really like type raising if you look at it closely.

<p>
<table>
<tr><td>this is the original: </td>
    <td><span style="color:red">λx. λy.
        <strong>like(</strong>y<strong>,
        </strong>x<strong>)</strong></span></td>
    <td> : t → t → f</td>
</tr>
<tr><td>this would be the type raised version: </td>
    <td><span style="color:red">λO. O@(λx. λy.
        <strong>like(</strong>y<strong>,
        </strong>x<strong>)</strong>)</span></td>
    <td> : ((t → t → f) → (t → f)) → (t → f)</td>
</tr>
<tr><td>this is what we used: </td>
    <td><span style="color:red">λO. λy. O@(λx.
        <strong>like(</strong>y<strong>,
        </strong>x<strong>)</strong>)</span></td>
    <td> : ((t → f) → f) → t → f</td>
</tr>    
</table>
</p>

Now we can see that the position of λy is what is different in the two
versions. So what does our version actually do? It's not that hard to
decipher. After a transitive verb is applied to an object, it is
supposed to return a predicate, that is a function which takes a term
and returns a formula stating something about the term.

Some examples of predicates we have seen:

  * <span style="color:red">λx. **growl(**x**)**</span> : t → f
  * <span style="color:red">λy. **like(**y**, <span
    style="color:green">mia</span>)**</span> : t → f
  * <span style="color:red">λy. <span style="color:orange">**∃y.
    (**<span style="color:teal">**woman(<span
    style="color:orange">y</span>)**</span> **∧** <span
    style="color:red">**like(**y**, <span
    style="color:orange">y</span>)**</span>**)**</span></span> : t → f
    
Let's dissect our "likes" term then.

  * <span style="color:red">λO. λy. O@(λx.
    <strong>like(</strong>y<strong>,
    </strong>x<strong>)</strong>)</span> : ((t → f) → f) → t → f

First off, we accept the object as an argument and bind its name to
the variable <span style="color:red">O</span>. Then we define the
predicate we will return. The name of the predicate's argument will be
<span style="color:red">y</span>, this is the variable that the
subject will be replacing with its term. Finally, how does the body of
the predicate look like? To find out, we ask the object to build it,
capturing any term it wants to use to represent itself in the relation
using the variable <span style="color:red">x</span>. When <span
style="color:red">O</span> is the meaning of "Mia", the object sends
<span style="color:green">**mia**</span> as the term to represent it
and returns the relation. When <span style="color:red">O</span> is a
quantified phrase like "a woman", it sends the logical variable <span
style="color:orange">**y**</span> as the value of <span
style="color:red">x</span> and it wraps the resulting relation in a
quantifier providing a range for <span
style="color:orange">**y**</span>.

### Wat? ###

OK, so things got pretty crazy pretty fast. How come we ended up
having to repeatedly juggle higher-order functions and raise types? We
can find an answer to this by inspecting the complexity
(intertwinedness) of the expressions we set out to generate.

To produce <span style="color:red">**growl(<span
style="color:blue">vincent</span>)**</span> : f, we just had to apply
the second-order function "growls" to the first-order constant
"Vincent". This was a fairly trivial thing as the representation of
"Vincent" is just plugged into a hole in the represenation of
"growls". Similarly goes for plugging "Vincent" and "Mia" into the
binary <span style="color:red">**like**</span> relation.

Things get more interesting when we try to generate something like
**<span style="color:green">∀x. (<span style="color:blue">boxer(<span
style="color:green">x</span>)</span> → <span
style="color:red">growl(<span
style="color:green">x</span>)</span>)</span>** : f. When we want to
combine "every" and "boxer", we ask "every" to start. "Every" lays
down the quantifiers and asks "boxer" to build the restricting
formula, which in turn asks "every" for the term to be filled in the
predicate (the variable <span style="color:green">**x**</span>). In
order to handle this case, we raised the NPs over VPs and ended up
with third-order items for the NPs in the lexicon.

Finally, to get quantified noun phrases working as objects of
transitive verbs, we had to fix our definition of "likes", which we
raised over the NPs and which made "likes" a fourth-order item in the
lexicon. Let's take a look at applying "likes" to an object such as "a
woman", <span style="color:red">λy. <span style="color:orange">**∃y.
(**<span style="color:teal">**woman(<span
style="color:orange">y</span>)**</span> **∧** <span
style="color:red">**like(**y**, <span
style="color:orange">y</span>)**</span>**)**</span></span> : t → f.
Here we ask "likes" to build a term, which begins by putting down the
lambda abstraction (<span style="color:red">λy.</span>), then passing
control to the object, in this case "a woman". The object lays down
any necessary quantifiers (**<span style="color:orange">∃y. <span
style="color:teal">woman(<span style="color:orange">y</span>)</span>
∧</span>**) and asks "likes" to build the consequent of the
quantifier. "Likes" will build the formula (<span
style="color:red">**like(**y, ...*)*</span>), but not without taking a
term to place in the relation from the object (<span
style="color:orange">**y**</span>). In other terms, the function
assigned by the lexicon to "likes" is a fourth-order function, which
calls the third-order function corresponding to "a woman", which calls
the second-order lambda function defined within "likes" which uses the
first-order argument x supplied to it by the object "a woman".

This kind of there-and-back-again exchange of control is enforced by
the interleaved syntactic structure of the FOPL formulas we want to
produce (the colors corresponding to the individual lexical items do
not form sovereign subtrees). Each of the two constituents to be
combined contains only the information strictly contained in its
subtree. To combine these two pieces of information into an elaborate
structure, the two lambda terms have to cooperate with each other,
exchanging continuation-like lambda functions.


One Last Jump
-------------

OK, we have gotten our heads around the recursions in the compositions
and our pyramid of types, but there is one more thing I'd like to do.

If we look at our theory, there are still some nasty spots left. First
off, the type of NP is VP → S, i.e. NPs consume VPs to produce
sentences. This sounds quite unintuitive from a linguistic standpoint.
We would much rather have a VP which satisfies its valency by
consuming an NP as its subject and forms a sentence. Also, there is
something off about how the lexical entry for "likes" treats its
object fundamentally differently from its subject. Finally, the notion
of who gets to be the functor seems kind of arbitrary. Within VPs, it
is the verb itself, the head of the phrase. Within NPs, it is the
determiner (which is also the head of its phrase, if you
[subscribe to the theory of generative grammar] [ChomskyStyle]).
However, within the top-level sentence, it is not VP, the head of the
sentence, which is the functor, but the subject NP.

  [ChomskyStyle]: http://www.youtube.com/watch?v=3Dw5opEYprQ

All of these problems are linked and we can solve them in one stroke
using a technique we have used several times before. Can you guess
which? Yep, it's type raising. Here are the new lexical entries for
our verbs.

  * "growls" ↦ <span style="color:red">λS. S@(λx.
    **growl(**x**)**)</span> : ((t → f) → f) → f
  * "likes" ↦ <span style="color:red">λO. λS. S@(λy. O@(λx.
    **like(**y**,** x**)**))</span> : ((t → f) → f) → ((t → f) → f) → f

In the case of the intransitive verb, we simply type raise the VP
using the general type raising rule we have used on proper nouns
before. For "likes", it is a little bit more complicated. "Likes" is a
transitive verb, that is, it is a function from an NP to a VP. So
first, we have to strip away the outer lambda abstraction to get the
VP body contained inside and then we mechanically apply the type
raising rule on that VP term, changing <span style="color:red">λy.
...</span> into <span style="color:red">λS. S@(λy. ...)</span>.

Let's have look at our new type mapping.

  * S = f
  * N = t → f
  * NP = (t → f) → f
  * VP = NP → S 
  * VT = NP → VP

Much better! The VP type now actually makes sense linguistically, the
functors line up with head constituents, the arguments of transitive
verbs are now handled uniformly and we have even made the transitive
and intransitive verbs look similar to each other.

### Minor Caveats ###

If you try to switch the order of <span
style="color:red">S@(λy.</span> and <span
style="color:red">O@(λx.</span> in the lexical entry for "likes" (or
any other transitive verb for that matter), you will generate a
different representation for sentences which have quantifiers in both
the subject and object. For example:

  * <span style="color:green">**∀x.
  (<span style="color:blue">boxer(<span
  style="color:green">x</span>)</span> → <span style="color:red"><span
  style="color:orange">∃y. (<span style="color:teal">woman(<span
  style="color:orange">y</span>)</span> ∧ <span
  style="color:red">like(<span style="color:green">x</span>, <span
  style="color:orange">y</span>)</span>)</span></span>)**</span> : f  
  * <span style="color:orange">**∃y.
  (<span style="color:teal">woman(<span
  style="color:orange">y</span>)</span> ∧ <span style="color:red"><span
  style="color:green">∀x. (<span style="color:blue">boxer(<span
  style="color:green">x</span>)</span> → <span
  style="color:red">like(<span style="color:green">x</span>, <span
  style="color:orange">y</span>)</span>)</span></span>)**</span> : f  

These different formulas actually correspond to two different readings
of the original sentence "Every boxer likes a woman" (scope
ambiguity). The top interpretation is the original one and corresponds
to the reading "For every boxer there is a specific woman that he
likes", while the bottom interpretation corresponds to the reading
"There is a (one) woman and every boxer likes her".

This ambiguity can be represented in our theory (maybe somewhat
crudely) by providing two different lexical entries for transitive
verbs, one for each permissible order of argument qualifiers.

Finally, we should also do something about the FOPL variables we
generate. Hard-coding <span style="color:green">x</span> into the
lexical entry for "every" and <span style="color:orange">y</span> into
the one for "a" doesn't seem like a good solution, if only because it
precludes us from using the same quantifier in the arguments of one
verb, e.g. as in "Every boxer likes every woman".


Further Reading
---------------

  1. Montague, Richard: English as a Formal Language
  2. Montague, Richard: The Proper Treatment of Quantification in
     Ordinary English
  3. Blackburn, Patrick and Bos, Johan: Representation and Inference in
     Natural Language

I haven't read any of these, which might explain my state of mild
confusion, though I plan to read all of them soon. 1. and 2. are
foundational papers in computational semantics which I expect will
cover most of what I showed here. 3. is one of the few textbooks in
the field.
