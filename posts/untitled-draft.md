---
title: The Working Hipster's Clojure
date: '2012-12-18'
description:
categories: 'clojure'
tags: []

type: draft
---

Lately I have been working on a project for my *Applications of NLP*
class here at Université de Lorraine. The project consists of
extracting a Lexicalized [Tree Adjoining Grammar][tag] from a small
corpus of sentences and their semantic representations and then using
that grammar to generate new sentences (or hopefully at least some of
the original sentences!) from new semantic representations.

  [tag]: http://en.wikipedia.org/wiki/Tree_adjoining_grammar

However, in order to extract a plausible Tree Adjoining Grammar from a
corpus, the trees have to have certain characteristics typical of the
ones produced by TAGs (the most important one being that arguments and
modifiers are never siblings). Sadly, the parser we used to process
our corpora (Stanford Parser) does not yield such pretty trees and we
were thus forced to postprocess them. Xia and Palmer propose an
algorithm for normalizing parse trees for TAG extraction (*From
Treebanks to Tree Adjoining Grammars*). Instead of implementing their
approach, we chose to write our own algorithm, since we had trouble
seeing how their algorithm systematically accounts for some of the
phrase trees in our data and since our fragment of English was quite
limited and so were the transformations we needed to perform.

We weren't sure how the algorithm would work exactly, but we managed
to go from nothing to a working solution in the space of one
afternoon. Our process was to start with a simple approach, run it on
the data, query the results to find potential problems, introduce
constraints in the algorithm to fix the problems we found and repeat
until all problems were resolved. After that, I extended the algorithm
to one more phenomena which we have discovered and tried to make it
fit for public display by replacing inside jokes with comments.

I wrote the program in Clojure and since I wanted to post something
about how I use the language, I decided to go with this. It's not a
pretty program by far, but throwaway code is code too, so here goes.
You can see the final "program" with all of the debugging scaffolding,
development code still in place and some documentation thrown in
[here](http://gist.github.com/4323107). The program isn't designed to
be compiled and run, but it is rather a collection of definitions
useful for building a program that solves my problem. Also, it is
dependent on having access to the data I run it on, so you might have
a hard time running it on your machines.

I will discuss some of its parts now but instead of focusing on the
algorithm itself, I will spend my time talking about some of Clojure's
features I ended up using (for dubious benefit, I admit) and which are
quite rare in mainstream languages. It will be namely logic
programming, metaprogramming support and dynamic scoping.

## Logic programming

At first I was very optimistic about finding a solution to the
problem. I imagined formulating the algorithm as a sequence of
tree-rewriting rules of a form like this:

    (VP
      X
      Y
      (PP Z W))
    ->
    (VP
      (VP
        X
        Y)
      (PP Z W))

The rad thing about rules like this is that they are dead simple to
implement using unification:

<script src="https://gist.github.com/4323409.js"></script>

What is nice is that I get to use unification for checking the input
template, capturing values from the input tree and also filling them
in to the output template. Sadly, such simple rules have shown to be
not flexible enough for our problem. Nevertheless, I still stuck with
the idea of using logic programming, my courage bolstered by having
just made it through [The Reasoned Schemer][trs] without any serious
head injury. I feel that it turned out to be a good choice, even
though I hit some rough spots with respect to termination.

  [trs]: http://mitpress.mit.edu/books/reasoned-schemer

<script src="https://gist.github.com/4323541.js"></script>

Here is a useful predicate I have defined along the way. It works just
like Prolog's ```append/3```, but is variadic, meaning it can take an
arbitrary number of arguments. This has proven to be very useful in
destructuring lists (see ```fix-flat-npo``` for example use) given that
the alternative (in both Prolog and "vanilla" core.logic) is to
introduce lots of new variables. A gotcha I encountered when writing
this predicate is that if you write it with the recursive call last,
you will generate a sequence of goals that in the case of the first
argument being a fresh variable never terminate after producing all
the valid solutions (Fun, right? OK, by now you might've guessed I'm
into this whole logic programming thing only because functional
programming was getting way too comfortable and mainstream).

One more example.

<script src="https://gist.github.com/4323622.js"></script>

The thing I like about the predicate above is the ```conde```
statement in the middle. ```conde``` is the core.logic way of writing
down disjunctions of goals, but it also has a lot to do with
```cond``` and branching. The interesting thing about this goal is
that it handles two cases which look quite different but share a lot
of the same logic. In a functional way, I would have to do some
branching to handle the shape of the input and extract the necessary
values from it (lines 12 and 14). Then I would have to do some general
computation with those values (all the lines outside of the
```conde```) and then branch again, constructing the correct output
for both of the input cases (lines 13 and 15). With logic programming,
I get the freedom of rearranging my statements without much
repercussions (see last example :-)). This allows me to take all my
branch dependent code (the input destructuring and output
construction) and put it in one place, evading the needlessly complex
double bifurcation one would most likely end up with in a functional
program.

## Metaprogramming support

In the last example, you might have noticed the strange ```^::rule```
incantation up in the function definition. What's up with that? Well,
in my algorithm, there are several ways you could take a phrase tree
and do some normalization on it. I define these rules as binary
relations between the original tree and the normalized tree. I then
want to traverse the input tree bottom-up and try to normalize every
node using any rule that fits. For that, I will need a relation that is
basically a disjunction over all the defined rules in my system.

One way to do this would be to have a place in your program where you
manually call all the rules you have defined one by one and then you
have to update this list of calls everytime you rename/add/remove a
rule. Another way could be to create a big data structure and store
all the rules inside of it as anonymous functions. Depending on your
language, this is probably not the most convenient way to write and
test functions. In Clojure, you can have the comfort of defining the
rules the way you would have naturally and also of not having to
maintain an extra list of calls to your rules, thanks to its
metaprogramming facilities. But before we go any further, let's look
at the strange series of glyphs that started this whole discussion.

In Clojure data structures (ergo, in Clojure code), you can annotate
collections and symbols with metadata. Metadata are simple Clojure
maps and you can associate them to an item such as a symbol by simple
putting ```^``` followed by a metadata map and the symbol you want to
annotate, as in ```^{:how-meta? so-meta} hipster-symbol```. A lot of
the times, you want metadata entries which serve as boolean flags and
so there is syntactic sugar for that, allowing you to write
```^:approved idol``` instead of ```^{:approved true} idol``` for
keywords such as ```:approved```. Finally, there is also syntactic
sugar for a keyword qualified in the current namespace. If the current
namespace is ```germany.berlin```, you can write ```::approved```
instead of ```:germany.berlin/approved```. So, by writing ```^::rule
fix-adjunctiono``` in my namespace ```grook.preprocess```, I am
basically saying ```^{:grook.preprocess/rule true} fix-adjunctiono```.
This way of prefixing symbols with modifiers might be familiar to you
from the access modifiers and type tags of the C language family and
indeed, Clojure uses this system for the very same purposes too. The
great difference between the hard-coded keywords of those languages
and the system used in Clojure is that the Clojure one is completely
extensible (just like XML, even with the proper namespacing).

OK, now back to the metaprogramming part. Whenever you evaluate a
```def``` form in Clojure, any metadata on the symbol naming the new
Var is inherited by the defined Var. This Var is then
[reified][reification] inside your program, meaning you can query and
work with it as you would with any other value in your program. So,
for example, we can write a macro that generates that long boring
disjunction for us (OK, the generated disjunction in my program is not
that long since in my program, I only have two rules, hence the
dubious benefit I talked about in the opening).

  [reification]: http://en.wikipedia.org/wiki/Reification_(computer_science)

<script src="https://gist.github.com/4323906.js"></script>

As a side note, if you want to use this in interactive development,
you have to be aware that simply removing a function definition and
recompiling your program will not remove previously defined Vars from
your running process, meaning that rules which you remove from your
source file will still apply. This can be easily fixed by clearing any
Vars defined as rules from your namespace before evaluating your
definitions (see top of the final program).

## Dynamic scoping


