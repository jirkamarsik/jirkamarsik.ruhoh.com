---
title: The Working Hipster's Clojure
tagline: Unpopular Techniques of Computer Programming
date: '2012-12-19'
description:
categories: 'clojure'
tags: ['core.logic', 'metaprogramming', 'dynamic-scope']

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
seeing how theirs offers a systematic account for some of the phrase
trees in our data and since our fragment of English was quite limited
and so were the transformations we needed to perform.

We weren't sure how the algorithm would work exactly, but we managed
to go from nothing to a working solution in the space of one
afternoon. Our process was to start with a naive program, run it on
the data, query the results to find potential problems, introduce
constraints in the algorithm to fix the problems we found and repeat
until all problems were resolved. After that, I extended the algorithm
to one more phenomena which we have discovered and tried to make it
fit for public display by replacing inside jokes with comments.

I wrote the program in Clojure and since I wanted to post something
about how I use the language, I decided to go with this. It's not a
pretty program by far, but throwaway code is code too, so here goes.
You can see the final "program" with all of the debugging scaffolding,
development code and some documentation thrown in
[here](https://gist.github.com/4323107). The program isn't designed to
be compiled and run, but it is rather a collection of definitions
useful for building a program that solves my problem. Also, it is
dependent on having access to the data I run it on, so you might have
a hard time running it on your machines.

I will discuss some of its parts now but instead of focusing on the
algorithm itself, I will spend my time talking about some of Clojure's
features and libraries I ended up using (for dubious benefit, I admit)
which are quite rare in mainstream languages. Namely, it will be logic
programming, metaprogramming facilities and dynamic scoping.

## Logic Programming

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
like Prolog's ```append/3```, but it is variadic, meaning it can take
an arbitrary number of arguments. This has proven to be very useful in
destructuring lists (see ```fix-flat-npo``` for
[example use][poly-appendo]) given that the alternative (in both
Prolog and "vanilla" core.logic) is to introduce lots of new
variables. A gotcha I encountered when writing this predicate is that
if you write it with the recursive call at the tail and then invoke it
with a fresh first argument, the solver will find all the correct
solutions, but when asked for more, it will never terminate (Fun,
right? OK, by now you might've guessed I'm into this whole logic
programming thing only because functional programming was getting way
too comfortable and mainstream).

  [poly-appendo]: https://gist.github.com/4323107#file-preprocess-clj-L120

One more example.

<script src="https://gist.github.com/4323622.js"></script>

The thing I like about the predicate above is the ```conde```
statement in the middle. ```conde``` is the core.logic way of writing
down disjunctions of goals, but it also has a lot to do with
[```cond```][cond] and branching. The interesting thing about this
goal is that it handles two cases which look quite different but share
a lot of the same logic. In a functional way, I would have to do some
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
program (another way to evade the double branching in a functional
program would be to abstract the common computations into functions,
but given the number of values that need to be computed and their
simplicity, this approach would have been quite cumbersome).

  [cond]: http://en.wikipedia.org/wiki/Cond

## Metaprogramming Facilities

In the last example, you might have noticed the strange ```^::rule```
incantation up in the function definition. What's up with that? Well,
in my algorithm, there are several ways you could take a phrase tree
and do some normalization on it. I define these rules as binary
relations between the original tree and the normalized tree. I then
want to traverse the input tree bottom-up and try to normalize every
node using any rule that fits. For that, I will need a relation that is
basically a disjunction over all the defined rules in my system.

One way to do this would be to have a place in your program where you
manually call all the rules you have defined one by one, but then you
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
maps and you can associate them to an item such as a symbol by putting
```^``` followed by a metadata map and the symbol you want to
annotate, as in ```^{:how-meta? so-meta} hipster-symbol```. A lot of
the times, you want metadata entries which serve as boolean flags and
so there is syntactic sugar for that, allowing you to write
```^:approved idol``` instead of ```^{:approved true} idol``` for any
keyword, such as ```:approved```. Finally, there is also syntactic
sugar for a keyword qualified in the current namespace. If the current
namespace is ```music.genres```, you can write ```::post-indie```
instead of ```:music.genres/post-indie```. So, by writing ```^::rule
fix-adjunctiono``` in my namespace ```grook.preprocess```, I am
basically saying ```^{:grook.preprocess/rule true} fix-adjunctiono```.
This way of prefixing symbols with modifiers might be familiar to you
from the access modifiers and type tags of the C language family and
indeed, Clojure uses this system for the very same purposes too. The
great difference between the hard-coded keywords of those languages
and the system used in Clojure is that the Clojure one is completely
extensible (just like XML, even with the namespaces
([code is data][homoiconicity])).

  [homoiconicity]: http://en.wikipedia.org/wiki/Homoiconicity

OK, now back to the metaprogramming part. Whenever you evaluate a
```def``` form in Clojure, any metadata you put on the symbol that
names the new Var is inherited by the defined Var itself. This Var is
[reified][reification] entity inside your program, meaning you can
query and work with it as you would with any other value in your
program. So, for example, we can write a macro that generates that
long boring disjunction for us (OK, the generated disjunction in my
program is not that long since in my program, I only have two rules,
hence the dubious benefit I talked about in the opening).

  [reification]: http://en.wikipedia.org/wiki/Reification_(computer_science)

<script src="https://gist.github.com/4323906.js"></script>

As a side note, if you want to use this in interactive development,
you have to be aware that simply removing a function definition and
recompiling your program will not remove previously defined Vars from
your running process, meaning that rules which you remove from your
source file will still apply. This can be easily fixed by clearing any
Vars defined as rules from your namespace before evaluating your
definitions (see
[top of the final program](https://gist.github.com/4323107#file-preprocess-clj-L8
)).

## Dynamic Scoping

During the development, we wanted an easy way to see what our program
was doing. Specifically, we wanted to have access to the constituents
it identified as adjuncts in the context of their parent phrases. I
could hardly imagine doing this in a purely functional setting, but
luckily, I had the freedom to switch my perspective from the
functional model of computation, which has helped me write my
algorithm, to the imperative model of computation, where I could start
thinking about how my program gets executed and insert the necessary
trace statements. To aggregate all the interesting values, I used a
global variable, more specifically an Atom, one of Clojure's reference
types.

It soon became obvious that inspecting the tree fragments by
themselves was not enough and that some link back to the original tree
would be necessary. This proved to be another tricky problem to solve
by pure functional programming. I have the names of the trees at one
place in the program but then I call some function which calls some
other function which calls another function which calls yet another
function from which I would then like to access the current tree name.
Ideally, I would do something like this, but due to the rules of
lexical scoping, it would fail.

<script src="https://gist.github.com/4329199.js"></script>

In imperative languages, this would be solved by storing the current
tree name in some ad-hoc instance or global variable. However, Clojure
offers us a slightly more elegant solution based on an alternative to
lexical scoping known as dynamic scoping.

One way to imagine the process of variable reference resolution in
lexical scoping is the following: when the program tries to resolve a
variable reference, it starts traversing the blocks of code
surrounding the variable reference inside-out until it finds one which
declares a variable having the same name. This means that variable
references can be resolved by just looking at the source code, making
the program easy to reason about. Lexical scoping is also one of the
key mechanisms in lambda calculus and gives us nice tricks like
closures. However, sometimes we might like to use a different
resolution strategy.

Contrast the above description of lexical scoping with that of dynamic
scoping: when the program tries to resolve a variable reference, it
starts traversing a stack of variable bindings top-to-bottom until it
finds one which declares a variable having the same name (you can
think of it sort of as the call stack, though function calls don't
have to correspond 1:1 to variable bindings). This means that when you
access a variable in dynamic scoping, you don't necessarily know where
it was bound to its current value. This kind of stuff is really wild,
but it can be quite useful in controlled doses.

<script src="https://gist.github.com/4329413.js"></script>

Here I define the dynamic (note the metadata keyword) Var
```*current-tree-name*```, which can be thought of almost as declaring
that the symbol ```*current-tree-name*``` will have dynamic scoping in
this namespace. The asterisks around the identifier are called
earmuffs and are a common convention when naming dynamically scoped
Vars. In every iteration of my tree processing loop, I bind the
dynamic Var to a new value, which pushes a new value on to the stack
of its bindings which is then accessed in ```record-adjunct```.

We could have done the same thing using a global variable, but even in
a simple scenario like this, we can see some advantages to using this
more high-level construct. The stack of dynamic bindings is
thread-local (as is the call stack) meaning that no other thread will
be affected by our new binding. Furthermore, once we leave the scope
of the binding, our value gets popped from the stack and the original
value is automatically restored, which prevents a class of bugs if we
would start manipulating the variable in nested contexts. In summary,
the value of ```*current-tree-name*``` after running our computation
stays the same as it was before and no one else can see a difference
in its value while it is running. Since I am using an Atom for the
global mutable list of performed adjunctions, I could run this code on
several threads at once without any possibility of race conditions and
without having to change the code to make it somehow explicitly
thread-safe.

Now, there is one more feature of dynamic Vars which I will reveal to
you and which will let us solve our problem in a cleaner manner by
getting rid of the global variable ```adjunctions```. When a
thread-local value has been bound to a dynamic Var, it is possible to
use the ```set!``` special form to modify its value. This might seem
like a cop out given Clojure's emphasis on immutability (it certainly
seems more like a part of its Lisp heritage than its lambda calculus
heritage), but it opens new doors for us. More specifically, it
enables a kind of bidirectional communication between functions
running on the call stack.

<script src="https://gist.github.com/4336670.js"></script>

This program does have some nice properties. We no longer have to
worry about the state of a global variable (which becomes a tangible
concern even in programs as simple as this if what you want is to keep
running new and new code live from the REPL). ```fix-tree``` now has a
nice interface. It performs the tree normalization and can also report
on the adjunctions it performed. Its docstring might say something
like this:

    If you bind a value to *fix-tree-adjunctions*, a record of every
    adjunction performed by fix-tree will be conjed onto it. If
    you also bind a value to *fix-tree-current-name*, it will
    incorporated into the generated records.

Now I can use ```fix-tree``` on arbitrary trees without it mutilating
a global variable that holds the results of my experiments. Parfait!

Well, not parfait yet. We can still make this simpler and therefore
prettier. Our habit of passing down the current tree name in a dynamic
Var came from the times when we used to store all of the traces in a
big global variable. Now it kind of sticks out and doesn't really make
sense. After all, all we really want to do is to just pass a trace of
the performed adjunctions back to the top-level caller.

<script src="https://gist.github.com/4336851.js"></script>

Now we have a function which doesn't muck up any shared state and
whose results are determined solely by its inputs. At the same time
though, it can easily yield a detailed log of its activity if the
caller opts in to it by binding a value to which the log should be
appended before calling the function proper. We managed that by just
hooking up our code with calls to a very brief logging function
instead of having to translate our code into some monad (if your code
already was in a monad and all you had to do was to apply a monadic
transformer, then more power to you :-)).

OK, that's all I've got today. You can find the full program at
[https://gist.github.com/4323107](https://gist.github.com/4323107).

## Further Reading

  * [Clojure Programming](http://www.clojurebook.com/): The most
    exhaustive book on Clojure I've read so far. Beatiful examples,
    practical under-the-hood details.
  * [The Joy of Clojure](http://joyofclojure.com/): The book that got
    me into Clojure, soon about to get the second edition treatment.
  * [The Reasoned Schemer](http://mitpress.mit.edu/books/reasoned-schemer):
    A lovely little book on building a tiny logic programming system
    in Scheme and implementing in it a relational version of the
    basics of Lisp and binary arithmetics. Serves as a very good
    introduction to the miniKanren system which is behind Clojure's
    logic programming library,
    [core.logic](https://github.com/clojure/core.logic).
